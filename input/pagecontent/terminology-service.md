<div>
	<blockquote>
		This content was previously published in the <a href="https://www.hl7.org/fhir/terminology-service.html">FHIR Terminology Service documentation</a>.
		It has been moved here to the Terminology Module Incubator to allow for more rapid iteration and development.
	</blockquote>

<a name="closure"></a>
<h3>Maintaining a Closure Table</h3>

<p>
The 5 operations Expand, Lookup, Validate, Subsumes, and Translate account for most operational requirements
associated with terminology use. However, there is one difficult but important use case that
they do not address, which is integrating terminologically based logic into application searches.
</p>
<p>
A typical example of this is a user that wants to find any observations for male patients over
the age of 50 who attended a particular clinic within a particular 2-week period, with a
diagnosis of gout, and who had an elevated serum creatinine.
</p>
<p>
In this case, both "diagnosis of gout" and "serum creatinine" involve value set and/or subsumption queries
(e.g. against SNOMED CT and LOINC respectively). This search has to be executed by some
logical processing engine that knows how to find patient related data in a given persistence store.
Often, this is some kind of SQL query, though many other technological choices are available.
However, this is done, the challenge with an operation like this is to integrate the
terminological knowledge into a search execution that also covers other relationships expressed in the search criteria.
</p>
<p>
One approach to this problem would be to use the expand operation above, so that the system executing
the search could generate expansions, and then search for these expansions. This has a couple of problems:
</p>
<ul>
  <li> the list of subsumed codes could be very long, and the search operation becomes correspondingly inefficient</li>
  <li> the expansion of the subsumption might not be closed, and so the search operation cannot be correct </li>
</ul>
<p>
An alternative approach is to generate a <a href="https://en.wikipedia.org/wiki/Transitive_closure#In_graph_theory">transitive closure table</a>
which lists all the possible transitive subsumption relationships, and <a href="http://karwin.blogspot.com.au/2010/03/rendering-trees-with-closure-tables.html">allows for rapid execution of these kind of queries</a> . However, this has other problems:
</p>
<ul>
  <li> the subsumption table can be very large (&gt;500000 records for SNOMED CT), even though very few of the codes are used</li>
  <li> subsumption tables are generally built up front, and do not deal with new codes as they are encountered very well</li>
  <li> they still do not offer a solution for non-closed expansions</li>
</ul>
<p>
This is the main reason why most systems do not support post-coordination or other forms
of coded expressions.
</p>
<p>
In FHIR, this problem is solved by building a closure table on the fly as new codes are seen.
This technique leaves the FHIR terminology server responsible for the terminological reasoning
and the client responsible for the closure table maintenance. To the client, it doesn't matter
whether the concept is post-coordinated or not. Here's a description of how the process works:
</p>
<ol>
 <li> the client defines a name associated with a particular context in which it wishes to maintain a subsumption based closure table</li>
 <li> the client registers this name with the FHIR Terminology server using the $closure operation (described in <a href="#closure">Maintaining a Closure Table</a> below)), with only one parameter, the name of the context</li>
 <li> any time the client system encounters a new Coding that is not entered in the closure table, it calls the $closure operation with the context name, and the Coding value it has encountered</li>
 <li> the server returns a ConceptMap resource with a list of new entries (code : system -&gt; code : system) that the client should add to its closure table
   <ul>
    <li> the server can indicate that entries should be removed from the table by providing a (code&nbsp;: system -&gt; code&nbsp;: system) with equivalence "unmatched" (though it's not known why that would be needed)</li>
   </ul>
 </li>
 <li> the client copies these entries into its closure table </li>
 <li> to facilitate the initialization process, a client can call $closure with multiple Coding values</li>
</ol>
<p>
The <a href="OperationDefinition-ConceptMap-closure.html">$closure operation</a> takes 2 parameters:
</p>
<ul>
 <li> closure table context name</li>
 <li> concepts to enter into the table (0 or more - 0 codings is a request to (re-)initialize the table)</li>
</ul>
<p>
The operation returns a concept map which has a list of mappings that represent new entries to make in the closure table.
The subsumption testing performed when building a closure table is the same as for the <code>$subsumes</code> operation, and is based on <a href="https://build.fhir.org/codesystem.html#subsumption">the CodeSystem definition of subsumption</a>.
</p>
<p>
The closure table can be resynchronized by passing an additional "version" parameter, which is a value taken from the
version in one of the delta responses. This is a request to replay all the mapping changes since that delta was sent.
</p>

<a name="closure-init"></a>
<h4>Initializing a Closure Table</h4>

<p>
Before it can be used, a closure table has to be initialized. To initialize a closure
table, POST the following to [base]/ConceptMap/$closure:
</p>
<div class="example">
<pre class="json">
{
  "resourceType" : "Parameters",
   "parameter" : [{
     "name" : "name",
     "valueString" : "[name]"
  }]
}
</pre>
</div>
<p>
A successful response is a 200 OK from the server, with an associated ConceptMap:
</p>
<div class="example">
<pre class="json">
{
    "resourceType": "ConceptMap",
    "id": "[name]",
    "version": "0",
    "name": "Closure Table [name] Creation",
    "status": "active",
    "experimental": true,
    "date": "2015-12-20T23:10:55Z"
}
</pre>
</div>
<p>
If there is an error (usually involving the closure name) the server returns a HTTP status 400 with an operation outcome:
</p>
<div class="example">
<pre class="json">
{
  "resourceType": "OperationOutcome",
  "text": {
    "status": "generated",
    "div": "&lt;div xmlns=\&quot;http://www.w3.org/1999/xhtml\&quot;&gt;&lt;p&gt;invalid closure name \&quot;invalid-id!\&quot;:&lt;/p&gt;&lt;/div&gt;"
  },
  "issue": [
    {
      "severity": "error",
      "details": {
        "text" : "invalid closure name \"invalid-id!\""
      }
    }
  ]
}
</pre>
</div>
<p>
What closure names are valid is at the discretion of the server.
</p>

<a name="closure-add"></a>
<h4>Adding to a Closure Table</h4>
<p>
When the consumer (client) encounters a new code, it POSTs the following to [base]/ConceptMap/$closure:
</p>
<div class="example">
<pre class="json">
{
  "resourceType" : "Parameters",
  "parameter" : [{
    "name" : "name",
    "valueString" : "[name]"
  }, {
    "name" : "concept",
    "valueCoding" : {
       "system" : "http://snomed.info/sct",
       "code" : "22298006",
       "display" : "Myocardial infarction"
    }
  }]
}
</pre>
</div>
<p>
Note that this example only includes one concept, but more than one is allowed:
</p>
<div class="example">
<pre class="json">
{
  "resourceType" : "Parameters",
  "parameter" : [{
    "name" : "name",
    "valueString" : "[name]"
  }, {
    "name" : "concept",
    "valueCoding" : {
       "system" : "http://snomed.info/sct",
       "code" : "22298006",
       "display" : "Myocardial infarction"
    }
  }, {
    "name" : "concept",
    "valueCoding" : {
       "system" : "http://snomed.info/sct",
       "code" : "128599005",
       "display" : "Structural disorder of heart"
    }
  }]
}
</pre>
</div>
<p>
The response varies depending on the conditions on the server. Possible responses:
If the closure table has not been initialized: Return a 404 Not Found with
</p>
<div class="example">
<pre class="json">
{
  "resourceType": "OperationOutcome",
  "text": {
    "status": "generated",
    "div": "&lt;div xmlns=\&quot;http://www.w3.org/1999/xhtml\&quot;&gt;&lt;p&gt;invalid closure name \&quot;[name]\&quot;:&lt;/p&gt;&lt;/div&gt;"
  },
  "issue": [
    {
      "severity": "error",
      "details": {
        "text" : "invalid closure name \"[name]\""
      }
    }
  ]
}
</pre>
</div>
<p>
If the closure table needs to be reinitialized:
Return a 422 Unprocessable Entity with
</p>
<div class="example">
<pre class="json">
{
  "resourceType": "OperationOutcome",
  "text": {
    "status": "generated",
    "div": "&lt;div xmlns=\&quot;http://www.w3.org/1999/xhtml\&quot;&gt;&lt;p&gt;closure \&quot;[name\&quot; must be reinitialized&lt;/p&gt;&lt;/div&gt;"
   },
   "issue": [{
       "severity": "error",
       "details": {
         "text" : "closure \"[name]\" must be reinitialized"
       }
     }
   ]
}
</pre>
</div>
<p>
The server should only send this when its underlying terminology conditions have been changed (e.g. a new version of SNOMED CT has been loaded). When a client gets this, its only choice is to initialize the closure table, and process all the codes in the closure table again (the assumption here is that the system has some external source of 'all the codes' so it can rebuild the table again).
If the concept(s) submitted are processed ok, but there's no new concepts, or no new entries in the table, return a 200 OK with :
</p>
<div class="example">
<pre class="json">
{
    "resourceType": "ConceptMap",
    "id": "[name]",
    "version": "[version]",
    "name": "Updates for Closure Table [name]",
    "status": "active",
    "experimental": true,
    "date": "2015-12-20T23:12:55Z"
}
</pre>
</div>
<p>
If there's new entries in the closure table, the server returns a 200 OK with:
</p>
<div class="example">
<pre class="json">
{
  "resourceType": "ConceptMap",
  "id": "b87db127-9996-4d0c-bda9-a278d7a24a69",
  "version": "[version]",
  "name": "Updates for Closure Table [name]",
  "status": "active",
  "experimental": true,
  "date": "2015-12-20T23:16:24Z",
  "group": [{
    "source": "http://snomed.info/sct",
    "target": "http://snomed.info/sct",
    "element" : {
      "code": "22298006",
      "target": [{
        "code": "128599005",
        "relationship": " source-is-narrower-than-target"
      }]
    }
  }]
}
</pre>
</div>
<p>Notes</p>
<ul>
 <li>The server can return multiple elements, each with 1 or more targets</li>
 <li>servers may return the relationship represented in either direction</li>
 <li>it's important to understand the relationship the right way around. From the definition for <a href="https://www.hl7.org/fhir/conceptmap-definitions.html#ConceptMap.group.element.target.relationship ">ConceptMap.group.element.target.relationship</a>: the relationship is read from source to target (e.g. source-is-narrower-than-target). So in this case, 128599005 (Structural disorder of heart) subsumes 22298006 (Myocardial infarction)</li>
 <li>In the $closure operation, the response never explicitly states that a code is subsumed by itself. Clients should assume that this is implicit</li>
 <li>The version is important. Each new invocation of the $closure operation returns a new version of the concept map. The server must keep track of the versions it has issued for replay (see below)</li>
 <li>As well as entering codes that are actually used, the client also enters search terms into the closure table</li>
 <li>The combination of the system and code is the key to the closure table; if the server encounters two different codes that have the same meaning (e.g. syntactical variation), it should create an "equals" relationship between them</li>
</ul>

<a name="closure-rerun"></a>
<h4>Re-running Closure operation</h4>
<p>
Given the way that the closure operation functions, it's possible for a client to lose a response from
the server before it is committed to safe storage (or the client might not have particularly safe storage).
For this reason, when a client is starting up, it should check that there have been no missing operations.
It can do this by passing the last version (from the Concept Map response) it is sure it processed in the request:
</p>
<div class="example">
<pre class="json">
{
  "resourceType" : "Parameters",
   "parameter" : [{
     "name" : "name",
     "valueString" : "[name]"
  }, {
     "name" : "version",
     "valueString" : "3"
  }]
 }
</pre>
</div>
<p>
That's a request to return all the additions to the closure table since version 3. The server returns its latest version in the concept map, along with anything added to the closure table since version 3 (not including version 3)
</p>
<p>
Notes:
</p>
<ul>
 <li>The client can pass a concept or version, but not both</li>
 <li>These examples use a serially incrementing sequential integer, 
   but this is not required, and clients should not assume that there is any meaning or order in the version. 
   Just recall the last version and treat it as a magic fixed value only meaningful to the server. There is, however, one special value: '0'. 
   Passing a last version of 0 should be understood as resyncing the entire closure table</li>
</ul>

<a name="closure-use"></a>
<h4>Making use of the Closure Table</h4>

<p>
The client uses the result of the closure operation to maintain a closure table. Simplistically,
it might look like this:
</p>
<table class="grid">
 <tr>
   <td><b>Scope</b></td>
   <td><b>Source</b></td>
   <td><b>Target</b></td>
   <td><b></b></td>
 </tr>
 <tr>
   <td>patient-problems</td>
   <td>http://snomed.info/sct|22298006</td>
   <td>http://snomed.info/sct|128599005</td>
 </tr>
 <tr>
   <td>patient-problems</td>
   <td>http://snomed.info/sct|24595009</td>
   <td>http://snomed.info/sct|90560007</td>
 </tr>
 <tr>
   <td>obs-code</td>
   <td>http://loinc.org|14682-9</td>
   <td>http://loinc.org|LP41281-4</td>
 </tr>
</table>

<p>
The client can then use a table like this as part of its general search conditions. Using the example from above:
"Find any observations for male patients over the age of 50 who attended a particular clinic within a particular
2-week period, with a diagnosis of gout, and who had an elevated serum creatinine." This query could be done, for
instance, with an SQL query like this:
</p>
<pre>
 Select * from Observations, Patients, Encounters, Conditions, Observations as Obs2 where
   Observations.patient = Patients.Key and Patients.Age > 50 and
   Observations.encounter = Encounters.Key and Encounter.clinic = [key]
     and encounter.date >= [date] and encounter.date &lt;= [date] and
   Conditions.patient = Patients.Key and Conditions.code
     in (<span style="color: DarkGreen">select Source From ClosureTable
       where Scope = "patient-problems" and Target = "http://snomed.info/sct|90560007"</span>) and
   Obs2.patient = Patients.Key and Obs2.value > 0.19 and Obs2.code
     in (<span style="color: DarkGreen">select Source From ClosureTable
       where Scope = "obs-code" and Target = "http://loinc.org|LP41281-4"</span>)
</pre>
<p>
Note that in real clinical systems, tables are usually far more structured than this
example implies, and the query is correspondingly more complex. The closure table
would usually be normalised - this example is kept simple to demonstrate the concept.
</p>
</div>
