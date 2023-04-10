# Streaming SSR with React 18

React 18 has introduced a lot of exiting changes and features.

React 18 SSR architecture. To understand the breakthoughs that react 18 brought, it's essential to look at the entire timeline and the increamental steps that led up to it.


before SSR

the reason why we do SSR in the first place. in particular we ;; dive into its importance and the aspects of it that shaped the way the react team decided to improve their SSR architecture.


The benifits of SSR:

* Performace
* Search engine optimization (SEO)
* User experience
  There exists a specific rendering flow of React application using SSR. First, the server takes over the client's responsibilityof fetching all the data and rendering the entire React application. after doing so, the resulting HYML and Javascript are sent from the server to the client. Lastly the client puts the html on the screen and connectss it with appropriate javascript which is also known as the hydration process. now the client receives the entire html structure instead of one enormous bundle of javascript that it needs to render itself.

the benifits if tgis flow include esasier access for web crawlers to index these page, which imporoves SEO and the cliejnt can quickly show the generated HTML to the user instead of a blank screenm which imporoves UX becaause a;; the rendering happenes on the serverm the client is relived of this duty and doesnt risk becoming a bottleneck in the scenaario of low end decies, leading to improved performance.


owever the setip as desscribed is only a. starting point for SSR. based on how thigs are implemented above, there is a lot more to gain in term of performance and UX, with these two aspects in mind let's tale a trip down React SSR memry lane, dive into the issue pre -react 18 experience its ebolition over time and learn jhow react 10 with its streaming features changed everything.



(Streaming) ssr, pre-react 18


before react 18, Suspense, or any of the new steraming feature existed the typical SSR setup in react would look something as follows. while different implementations will probably contain minor differences, most setup will follow a similar architecture.




`// server/index.ts`

```
import path from 'path'
import fs from 'fs

import React from 'react'
import ReactDOMServer from 'react-dom/server'
import express from 'express'

import {APP} from 'client/app'

const app = express();

app.get('/', (req,res) => handleGetRequest)

function handleGetRequest(req, res){
const appContent = ReactDOMServer.renderToString(<App/>)
const indexFile = path.resolve('build/index.html');
fs.readFile(indexFile, 'utf8', (err, data) =>{
if(error){
// log the error
	return res.status(RESCODES.InternalError).send(RESERRORMS.failedToLoad);
}
return res.send(data.replace('<div id="root"></div>', `<div id="root">${app}</div>`));
})

}
app.use(express.static('build'));


```



The biggest part of an SSR setup is the sever, so lets start with thtat in this example, we're using express to spin up a server to serve the files from our build folder on port 8080. when the server receives a request at the root URL it will rende the React aplication to an HTML string usng the renderTOString function from the ReactDOMServer pacakge.

The result needs to be sent back to the client. but before that the srver needs to surround the rendered application with the approproate HTML structure. to do so, this example looks in the build folder for the index.html file, import it and injects the rendered application into the root element:

```
//Client/index.ts

import React from 'react'
import ReactDOM from 'React-dom'
import App from './app'

//instead of ReacrDom.render()
ReactDOM.hydrate(<App/>, doc.getID)

```

Then the main change that needs to be made on the client side is that it doesnt have to render the application anymore.


As we saw in the previous step the application is already rendered by the server. so now the client is onl responsible for hydrating the application. it does so by using the ReactDOM.hydrate function instead of ReactDom.render


while is a working setup of React SSR, there are still few major drawbacks to it regarding preformance and UX:

* while the server is now resposible for rendeing the React application, the server-side-rendered content is still one large blob of the HTML that needs to be transmitted toward the cloent before it's rendered
* Due to the interdependent nature ofReact components, the server must wait for all the data to be featched before it can start rendering components, generate the HTML esponse, and send it to the client
* The client still needs to load the entire appp's javascript before it can start hydrating the server's HTML response
* the hydration process is something that needs to happen all at once, but components are only inteactive after being hydrated, which means that users cannot interact with a page before hydration is complete

  In the end all of thise drawbacks boil down to the current setup, which is still a waterfall-like approach from the sever towards the client. this creates an all-or-nothing flow from one end to the other end: either the entire HTML response is sent to the clinet or not, either all the data is done fetching so the server can start rendering or not, either way the entire application is hydrated or not and either the entire page is responsive or none.

In React 16, the renderToNodeSteam server rendering function was introduced on top of the existing renderToString. in terms of the setup and results, this didn;t change a lot except that this function retuns a Node.js ReadableStream. this alloes the server to stream the HTML to the client.

```
app.get('/', handleGetRequest())

function handleGetRquest(req, res){
const endHTML = "</div></body></html>"
const indexFile = path.resolve('build'/index.html)

fs.readFile(indexFile, 'utf8' (err,data) => {
	if(err){
	return res.status(RESCODE.InternalError).send(RESMES.InternalError);
}

// Split the html where the react app should be injected and send
const beginHTML = data.replace('<html><head><body>','')
res.write(beginHTML)

// Render the application into a stream using `renderToNodeStream` 
const appStream = ReactDOMServer.renderToNodeStream(<App />);
appStream.pipe(res, {end:'false'});

const endHTML = "</div></body></html>";
appStream.on('end', () => response.end(endHTML);

});

}
```

This new function partly solves one of the drawbacks we described; namely it has to ransmit the HTML response as one large blob from the server to the clinet. however the server still needs to wiat for the entire HTML structure to be grerated before it can start transmitting anything to the client. so it doesn't really address any of the other drawbacks the we described.


Now lets look at the situtation after React 18 with newlu introducd features and how they address these drawbacks.


Streaing SSR post-React 18

The SSR architechture post-React 18 invloced a handfu; of different parts. None of these single-handedly fixed any of the drawbacks that we described, but the combination of them makes the magiv work. so to fully nderstand the entire setup, its necessary to look into all of them and what they contribute.


the Suspese component:

At the center of it all is the famous suspense somponent. it is the main gateway towards all features that we'll describe, so let's start with it.


```
import {lazy,Suspense} from 'react';
const SomeInnercomp = lazy(() => 
import('./someInnercomp' /* webpackPrefetch:true */)) 
export deafult function someComp(){
 return (<Suspense fallback={<Fallback />}>
	<SomeInnerComp />
</Suspense>
)}

```

In Short Suspense is a machanisum for developers to tell React that a certain part of the application is waiting for data. In the meantime React will show a fallback UI in its place and update it when the data is ready.


This doesn't sound too different from previous approaches, buut fundamentally, it synchronizes React's rendering proccess and the data-featching process in a way that is more graceful and integrated.


Suspense boundaries split up the application into chhunks based on their data feathcing requirements, which the sever can then use to delay rendering what is pending. meanwhile ,it can pre-render the chunks for which data is available and strea, it to the client. when the data for a previously pending chunk is ready, the server will then render it and send it the client again using the open stream.


Togather with the React Lazy which is used to code split your javascript bundle into smaller parts it provides the first pieces of the puzzle towards fixing the remaining waterfall drawbacks.

The problem was that the Suspense and code-splitting using react lazy were not compatible with SSR yet, until React 18.


The renderTopipeableStream API:

To understand the remaining connecting puzzle pieces, we'll take alook at the suspense SSR example that the react teams provided in their working groip discussion for the architecture ppost React 18.


```
import ReactDOMServer from 'React-dom/server';
import {App} from './App'
app.get('/', handleGetRequest())
function handleGetRequest(req, res){
res.socket.on('error', (err) => //logErr)

let didError = false;
const stream = ReactDOMServer.renderTopipealbeStream(<App/>, {
	bootstrapScripts:[/main.js'],
	onShellready:() => {
	res.statusCode = didError ? 500 : 200;
	res.setHeader('Content-type', 'text/html');
	stream.pipe(res);
	}
	onError:(err) => {
	didError = true
	}
})
}

```


The most significant change compared to the previous setup is the usage of renderToPipeableStream API on the server side, this is a newly introduced server rendering function in React 18, which retruns a pipeable Node.js stream. while the previous renderToNodeStream couldn't wait for the data and would bugger the entire HTML content util the end of the stream, the renderToPipeableStrea function does not suffer from these limitations.


when the content above the Suspense boundary is ready, the onShellReady callback is called, if any error happened in themeanwhile, it'll be reflected that int the response towaeds the client. then , we'll start streaming the HTML to the client by piping it into the response.

after the stream will stay open and transmit any subsequent rendered HTML blocks ti the client. this is the biggest change compared to its former version

This rendering function fully integrates with the suspense feature and coe splitting through Raect.lazy on the side of the server, which is what enables the streaming HTML feature for SSR. this solves the previously described waterfall of both HTML and data featching, as the applicatio can be rendered and transmitted incrementally basedon data requirements.


```
//clinet/index.ts
import React from "react"
import RactDOMClient from "react-dom/client"";
improt {App} from "./App"

ReactDOMClient.hydrateRoot(doc.id, <App/>);
```


Introducing ReactDOMClient.hydrateRoot for selective hydration


on the client side, the only change that needs to be made is how. the application is put on the screen. As a replacement for the previous ReactDOM.hydrate, the React team introduced new ReactDOMClinet.hydrateRoot in React 18 while the changes is minimal, it enables a lot of improvements and features for our context, the most important one is selective hydration.


As mentioned, Suspense splits the application inot HTML chunks on data requirements while code splitting splits the application into javascript chunks. Selective hydration will allow React to put these things togather on the client and start hydrating chunks at different timings and priorities. it can start hydrating as soon as chunks of HTML and JS are received, and prioritize a hydration queue of parts that the user interacted with.


This solves the remaining two waterfall issues that we had: having to wait for all javascfipt to load before hydrating can start, and. either hydrating the entire appliation or none or if
