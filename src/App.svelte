<script>
	const myEngine = Comunica.newEngine();

	let data = parse(document.body.getAttribute('cv'));

	async function parse(url) {
		const person = await parsePerson(url);
		const seeAlso = await parseSeeAlso(url);
		const workHistory = await parseWorkHistory(url);
		const education = await parseEducation(url);
		const other = await parseOther(url);

		person['seeAlso'] = seeAlso;
		person['workHistory'] = workHistory;
		person['education'] = education;
		person['other'] = other;

		console.log(person);

		return person;
	}

	async function parseOther(url) {
		const result = await myEngine.query(`
			PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
			PREFIX cv: <http://rdfs.org/resume-rdf/cv.rdfs#>
			SELECT 
				?otherInfoDescription
				?otherInfoType
			WHERE {
				?id cv:hasOtherInfo/rdf:rest*/rdf:first ?o .
				?o cv:otherInfoDescription ?otherInfoDescription ;
				   cv:otherInfoType ?otherInfoType .
			}
		`, {
			sources: [url],
		});

		const bindings = await result.bindings();

		let seeAlso = [];

		bindings.forEach( b => {
			seeAlso.push({
				otherInfoDescription: mayBe(b,'otherInfoDescription'),
				otherInfoType: mayBe(b,'otherInfoType') ,
			});
		});

		return seeAlso;
	}

	async function parseEducation(url) {
		const result = await myEngine.query(`
			PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
			PREFIX cv: <http://rdfs.org/resume-rdf/cv.rdfs#>
			SELECT 
				?eduDescription
				?startDate
				?endDate
			WHERE {
				?id cv:hasEducation/rdf:rest*/rdf:first ?o .
				?o cv:eduDescription ?eduDescription.
				OPTIONAL { ?o cv:eduStartDate ?startDate . }
				OPTIONAL { ?o cv:eduEndDate ?endDate . }
			}
		`, {
			sources: [url],
		});

		const bindings = await result.bindings();

		let seeAlso = [];

		bindings.forEach( b => {
			seeAlso.push({
				eduDescription: mayBe(b,'eduDescription'),
				startDate: mayBe(b,'startDate') ,
				endDate: mayBe(b,'endDate') ,
			});
		});

		return seeAlso;
	}

	async function parseWorkHistory(url) {
		const result = await myEngine.query(`
			PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
			PREFIX cv: <http://rdfs.org/resume-rdf/cv.rdfs#>
			SELECT 
				?jobType 
				?jobDescription
				?startDate
				?endDate
			WHERE {
				?id cv:hasWorkHistory/rdf:rest*/rdf:first ?o .
				?o cv:jobType ?jobType ;
				   cv:jobDescription ?jobDescription.
				OPTIONAL { ?o cv:startDate ?startDate . }
				OPTIONAL { ?o cv:endDate ?endDate . }
			}
		`, {
			sources: [url],
		});

		const bindings = await result.bindings();

		let seeAlso = [];

		bindings.forEach( b => {
			seeAlso.push({
				jobType: mayBe(b,'jobType') ,
				jobDescription: mayBe(b,'jobDescription'),
				startDate: mayBe(b,'startDate') ,
				endDate: mayBe(b,'endDate') ,
			});
		});

		return seeAlso;
	}

	async function parseSeeAlso(url) {
		const result = await myEngine.query(`
			PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
			PREFIX schema: <http://schema.org/>
			SELECT ?label ?url 
			WHERE {
				?id rdfs:seeAlso/rdf:rest*/rdf:first ?o .
				?o rdfs:label ?label ;
				   schema:url ?url.
			}
		`, {
			sources: [url],
		});

		const bindings = await result.bindings();

		let seeAlso = [];

		bindings.forEach( b => {
			seeAlso.push({
				label: mayBe(b,'label') ,
				url: mayBe(b,'url'),
			});
		});

		return seeAlso;
	}

	async function parsePerson(url) {
		const result = await myEngine.query(`
			PREFIX schema: <http://schema.org/> 
			PREFIX foaf: <http://xmlns.com/foaf/0.1/>
			PREFIX vcard: <http://www.w3.org/2006/vcard/ns#>
  			SELECT 
			  	?name 
				?description 
				?image 
				?title 
				?affiliation
				?workplaceHomepage 
				?email
			WHERE {
				?id a schema:Person ;
					schema:name ?name ;
					schema:description ?description ;
					schema:affiliation ?affiliation ;
					schema:email ?email ;
					foaf:workplaceHomepage ?workplaceHomepage ;
					vcard:title ?title ;
					foaf:img ?image .
  			} `, {
  			sources: [url],
		});

		const bindings = await result.bindings();

		const b = bindings[0];

		return {
			name: mayBe(b,'name'),
			description: mayBe(b,'description') ,
			image: mayBe(b,'image') ,
			title: mayBe(b,'title') ,
			affiliation: mayBe(b,'affiliation') ,
			workplaceHomepage: mayBe(b,'workplaceHomepage') ,
			email: mayBe(b,'email'),
		}
	}

	function mayBe(binding,name) {
		return binding.get('?' + name)?.value;
	}
</script>

<svelte:head>
  {#await data}
  {:then cv}
   <title>CVViewer - {cv.name}</title>
  {/await}
</svelte:head>

{#await data}
{:then cv}
<div class="container-fluid">
	<div class="row content">
	  <div class="col-sm-3 sidenav">
		<h4>{cv.name}</h4>
		<h5>{cv.title}</h5>
		<ul class="nav nav-pills nav-stacked">
		  <li class="active"><a href="#section1">Home</a></li>
		  <li><a href="{cv.affiliation}">Affiliation Homepage</a></li>
		  <li><a href="{cv.workplaceHomepage}">Work Homepage</a></li>
		</ul><br>
		<div class="input-group">
		</div>
	  </div>
  
	  <div class="col-sm-9">
		<h4><small>CURRICULUM VITAE</small></h4>
		<hr>
		<img src="{cv.image}" alt="image of {cv.name}"/>
		<h2>About me</h2>
		<p>{cv.description}</p>
		
		<hr>
		<h2>Contact</h2>
		<p>{cv.email}</p>

		<hr>
		<h2>See Also</h2>
		{#each cv.seeAlso as seeAlso}
			<p><b>{seeAlso.label}</b> <a href="{seeAlso.url}">{seeAlso.url}</a></p>
		{/each}

		<hr>
		<h2>Work History</h2>
		{#each cv.workHistory as work}
			<h4>
				{work.startDate.substring(0,4)} -
				{#if work.endDate} 
					{work.endDate.substring(0,4)}
				{:else}
					NOW
				{/if}
				:
				{work.jobType}
			</h4>
			<p>{work.jobDescription}</p>
		{/each}

		<hr>
		<h2>Other experience</h2>
		{#each cv.other as other}
			<p>
				<b>
				{other.otherInfoType}
				</b>
				:
				{other.otherInfoDescription}
			</p>
		{/each}

		<hr>
		<h2>Education</h2>
		{#each cv.education as edu}
			<p>
				<b>
				{#if edu.endDate} 
					{edu.endDate.substring(0,4)}
				{/if}
				</b>
				:
				{edu.eduDescription}
			</p>
		{/each}

	  </div>
	</div>
  </div>
  
  <footer class="container-fluid">
	<p>Generated with <a href="https://github.com/phochste/CVViewer">CVViewer</a></p>
  </footer>

{/await}

<style>
    /* Set height of the grid so .sidenav can be 100% (adjust if needed) */
    .row.content {height: 1500px}
    
    /* Set gray background color and 100% height */
    .sidenav {
      background-color: #f1f1f1;
      height: 100%;
    }
    
    /* Set black background color, white text and some padding */
    footer {
      background-color: #555;
      color: white;
      padding: 15px;
    }
    
    /* On small screens, set height to 'auto' for sidenav and grid */
    @media screen and (max-width: 767px) {
      .sidenav {
        height: auto;
        padding: 15px;
      }
      .row.content {height: auto;} 
    }
  </style>