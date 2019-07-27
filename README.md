# Solid-Pod-Data-Browser-Search
Use the Solid Data Browser app to search and view any solid pod url or uri. Simply instert the Solid Pod address ie., https://example.solid.community/public into the Solid interactive Data Browser search bar and click "search". Your results will appear in a Solid Data Browser Window. There is also a feature of a html, css and js code editor.
<!DOCTYPE html>
<html id="docHTML">
<head>
  <title>Solid Pod Data Browser</title>
  <meta content="text/html; charset=UTF-8" http-equiv="content-type">
  <!-- was https://solid.github.io/solid-panes/style/tabbedtab.css   -->
  
    <link type="text/css" rel="stylesheet" href="https://timbl.com/timbl/Automation/Library/Mashup/mash.css" />
    <script type="text/javascript" src="https://timbl.com/timbl/Automation/Library/Mashup/mashlib.js"></script>

<script>
document.addEventListener('DOMContentLoaded', function() {
    const panes = require('mashlib')
    const UI = panes.UI
    const $rdf = UI.rdf
    const dom = document
    $rdf.Fetcher.crossSiteProxyTemplate = self.origin + '/xss?uri={uri}';
    var uri = window.location.href;
    // window.document.title = 'Data browser: ' + uri;
    var kb = UI.store;
    var outliner = panes.getOutliner(dom)

    function complainIfBad (ok, message) {
      if (ok) return
      let msg = box.appendChild(dom.createElement('p'))
      msg.textContent = message
    }

    function expandDataBowser ( event ) {
      let uri = $rdf.uri.join(subjectEle.value, window.location.href)
      console.log("User field " + subjectEle.value)
      console.log("User requests " + uri)

      const params = new URLSearchParams(location.search)
      params.set('uri', uri);
      window.history.replaceState({}, '', `${location.pathname}?${params}`);

      var subject = kb.sym(uri);
      // UI.widgets.makeDraggable(icon, subject) // beware many handlers piling up
      outliner.GotoSubject(subject, true, undefined, true, undefined);
    }

    async function applyForm ( event ) {
      let uri = $rdf.uri.join(subjectEle.value, window.location.href)
      let subject = $rdf.sym(uri)
      console.log("Subject field " + subjectEle.value)
      console.log("Loading subject: " + subject)
      await kb.fetcher.load(subject)
      console.log("Loaded subject. ")

      let formURI = $rdf.uri.join(formEle.value, window.location.href)
      let form = $rdf.sym(formURI)
      console.log("Form field " + formEle.value)
      console.log("Loading form: " + form)
      await kb.fetcher.load(form)
      console.log("Loaded form. ✅")

      console.log('Check predicates in form...')
      const predicates = kb.statementsMatching(null, UI.ns.ui('property')).map(st => st.object)
      var ontologyURIs = {}
      for (var pred of predicates) {
        console.log('        predicate: ' + pred)
        let nsuri = pred.doc().uri
        ontologyURIs[nsuri] = true
      }
      for (let ontURI in ontologyURIs) {
        let ontology = $rdf.sym(ontURI)
        ontURI = ontURI.replace(/^http:/, 'https:') // hack for browser security -- sigh
        if (ontURI.startsWith('https://purl.org/dc/terms/')) {
          console.log('   Using W3C archive version of ' + ontURI)
          ontURI = ontURI.replace(/^https:\/\//, 'https://www.w3.org/archive/')
          // ontURI = 'https://www.w3.org/archive/purl.org/dc/terms/ontology.rdf'
        }
        addToTray(ontology)
        console.log('Loading ontology URI ' + ontURI)
        let options = { withCredentials: false} // sigh more browser security
        try {
          await kb.fetcher.load(ontURI, options) // or could pass array ontologyURIs.keys
        } catch (err) {
          complainIfBad(false, ontURI + ' ' + err)
        }
      }

      const params = new URLSearchParams(location.search)
      params.set('subject', subject.uri);
      params.set('form', form.uri);
      window.history.replaceState({}, '', `${location.pathname}?${params}`);

      let store = subject.doc() // @@ need footprints, allow user cotrol of store

      UI.log.debug = console.log // @@ for testing

      UI.widgets.appendForm(dom, box, {}, subject, form, store, complainIfBad)

      // UI.widgets.makeDraggable(icon, subject) // beware many handlers piling up
      // outliner.GotoSubject(subject, true, undefined, true, undefined);
    }

    async function addToTray (thing) {
      console.log('Add to tray:' + thing)
      function deleteThis () {
        // @@ logical removal from tray collection
        tray.removeChild(card)
      }
      var already = {}
      for (var ele of tray.children) {
        if (ele.subject) {
          already[ele.subject.uri] = true
        }
      }
      if (already[thing.uri]) {
        console.log('    dropped object already in tray. ignore.' + thing)
        return
      }

      var tr = UI.widgets.personTR(dom, null, thing, { deleteFunction: deleteThis}) // @@ add delete
      var card = dom.createElement('table') // its own
      card.appendChild(tr)

      card.subject = thing
      card.style.backgroundColor = '#8e8'
      card.style.borderRadius = '1em'
      card.style.margin = '0.3em'
      tray.appendChild(card)
    }

    async function handleURIsDroppedOnTray (uris) {
      for (uri of uris) {
        console.log('dropped uri: ' + uri)
        await addToTray($rdf.sym(uri))
      }
    }

    const box = dom.getElementById('box')
    const formEle = dom.getElementById('form')
    const subjectEle = dom.getElementById('subject')
    const goButton = dom.getElementById('goButton')
    const applyButton =  dom.getElementById('applyButton')
    /*
    subject.addEventListener('keyup', function (e) {
      if (e.keyCode === 13) {
        expandDataBowser(e)
      }
    }, false)
    */

    const tray = box.appendChild(dom.createElement('div'))
    tray.style = 'background-color: #ccc; padding: 1em; border-radius: 1em; min-width:30em; min-height: 5em;'
    UI.widgets.makeDropTarget(tray, handleURIsDroppedOnTray)

    // Basic stuff:

    goButton.addEventListener('click', expandDataBowser, false);
    applyButton.addEventListener('click', applyForm, false);

    let initialSubject = new URLSearchParams(self.location.search).get("subject")
    if (initialSubject) {
      subjectEle.value = initialSubject
    }
    let initialForm = new URLSearchParams(self.location.search).get("form")
    if (initialForm) {
      formEle.value = initialForm
    } else {
      // formEle.value = 'https://index,solid.community'
    }

    goButton.addEventListener('click', expandDataBowser, false);
    let initial = new URLSearchParams(self.location.search).get("uri")
    if (initial) {
      subject.value = initial
    } else {
      console.log('ready for user input')
      // subjectEle.value = 'https://index.solid.community' // @@ testing
    }

    async function main () {
      await addToTray($rdf.sym('https://index.solid.community'))
      await addToTray($rdf.sym('https://index.solid.community/public'))
      await addToTray($rdf.sym('https://index.solid.community/profile/card'))
      await addToTray($rdf.sym('https://yngwiemalmsteen.solid.community'))
      await addToTray($rdf.sym('https://yngwiemalmsteen.solid.inrupt.net'))
      await addToTray($rdf.sym('https://forums.solid.community'))
     
      await addToTray($rdf.sym('https://portal.solid.community'))
    }

    if (initialSubject && initialForm) {
      console.log('All parms in URL search -> do immediately')
      applyForm()
    } else {
      console.log('ready for user input')
    }
    main()
});
</script>
</head>
<body>
  <table style="width:100%;">
    <h2>Solid Pod Data Browser Search</h2>
    
    <tr style="font-size:100%">
      <td style="padding:1em; width:5em;" id="icon">Enter URL:</td>
      <td><input id="subject" type="text" style="font-size:100%; min-width:30em; padding:0.5em; width:95%;"/></td>
      <td  style="width:5em;"><input type="button" id="goButton" value="Search" /></td>
    </tr>
    <tr style="font-size:100%">
      <td style="padding:1em; width:5em;" id="icon">Form to use:</td>
      <td><input id="form" type="text" style="font-size:100%; min-width:30em; padding:0.5em; width:95%;"/></td>
      <td  style="width:5em;"><input type="button" id="applyButton" value="Get" /></td>
    </tr>
    <tr><td colspan="3">
      <div class="TabulatorOutline" id="box">
          <table id="outline"></table>
      </div>
    </td>
    </tr>
</table>
  <button type="button" onclick="alert('Thank you for checking out the Solid Pod Search App Please email me with your feedback solid@inrupt.community')">Feedback</button>

  </p>
  <h1>Code Editor</h1>

<iframe></iframe><body id=e><script>for(i=4;--i;)e.innerHTML+="<textarea id=t"+i+" placeholder="+[,"JS","CSS","HTML"][i]+" rows=9 onkeydown='if((K=event).keyCode==9){K.preventDefault();s=this.selectionStart;this.value=this.value.substring(0,this.selectionStart)+\"\t\"+this.value.substring(this.selectionEnd);this.selectionEnd=s+1}'>"+(unescape((l=location).hash.slice(1,-1)).split("\x7F")[i-1]||"");onload=onkeyup=function(a){q=[(E=escape)(j=t1[v="value"]),E(c=t2[v]),E(h=t3[v])].join("\x7f")+1;(H=history)&&H.replaceState?H.replaceState(0,0,"#"+q):location.hash=q;I=h||c||j?h+"<script>"+j+"<\/script><style>"+c:"<pre>Result";navigator.userAgent.match(/IE|Tr/)?((D=e.lastChild.contentWindow.document).write(I),D.close()):frames[0].location.replace("data:text/html,"+escape(I))}</script>



</body>
</html
