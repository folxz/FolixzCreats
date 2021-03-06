html {
	box-sizing: border-box;
}

*,
*:before,
*:after {
	box-sizing: inherit;
}

body {
	margin: 0;
	background: #000!important;
}

#gameContainer {
	width: 100vw;
	height: 100vh;
	background: #000!important;
}

canvas {
	width: 100%;
	height: 100%;
	display: block;
}
/* try to handle mobile dialog */

canvas + * {
	z-index: 2;
}

.logo {
	display: block;
	max-width: 15vw;
	max-height: 15vh;
}

.progress {
	margin: 1.5em;
	border: 1px solid white;
	width: 30vw;
	display: none;
}

.progress .full {
	margin: 2px;
	background: white;
	height: 1em;
	transform-origin: top left;
}

#loader {
	position: absolute;
	left: 0;
	top: 0;
	width: 100vw;
	height: 100vh;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
}

.spinner,
.spinner:after {
	border-radius: 50%;
	width: 5em;
	height: 5em;
}

.spinner {
	margin: 10px;
	font-size: 10px;
	position: relative;
	text-indent: -9999em;
	border-top: 1.1em solid rgba(255, 255, 255, 0.2);
	border-right: 1.1em solid rgba(255, 255, 255, 0.2);
	border-bottom: 1.1em solid rgba(255, 255, 255, 0.2);
	border-left: 1.1em solid #ffffff;
	transform: translateZ(0);
	animation: spinner-spin 1.1s infinite linear;
}

@keyframes spinner-spin {
	0% {
		transform: rotate(0deg);
	}
	100% {
		transform: rotate(360deg);
	}
}

.ad {
	position: absolute;
	background: rgba(0, 0, 0, 0.4);
	overflow: hidden;
	z-index: 40;
	display: none;
}

.modal{
	background:rgba(0,0,0,.4);
	display:none;
	height:100%;
	width: 100%;
	position:fixed;
	z-index:10000;
	top: 0;
	left: 0;
	bottom: 0;
	right: 0;
}

.modalContent{
	margin: auto;
	width: 100%;
}

.centered {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* The Close Button */
.close {
  color: #aaa;
  float: right;
  font-size: 28px;
  font-weight: bold;
}

.close:hover,
.close:focus {
  color: black;
  text-decoration: none;
  cursor: pointer;
}

#continueWindow{
	background-color: #fefefe;
	margin: 15% auto;
	padding: 20px;
	border: 1px solid #888;
	width: 30%;
}

#adWindow{
	background: #4382f5;
	border: 10px solid #4382f5;
	width: 660px;
	border-top: 0;
	height: 540px;
}
var tempErrorCreds;
var tempProviderName;

function retrieveIdToken(successCallback, errorCallback) {
	if(firebase.auth().currentUser === null){
		if(errorCallback !== null) 
			errorCallback("User is null");
		return;
	}

	firebase.auth().currentUser.getIdToken().then(function (idToken) {
		var resultObj = {
			token: idToken,
			displayName: firebase.auth().currentUser.displayName
		};

		if (successCallback !== undefined) {

			successCallback(resultObj);
		}
	})
		.catch(function (error) {
			console.log(error);
			if (errorCallback !== undefined)
				errorCallback(error.message);
		});
}

function anonymousLogin(successCallback, errorCallback) {
	var resultObj = {
		token: "",
		displayName: "guest"
	};

	if (successCallback !== undefined) {

		successCallback(resultObj);
	}
}

function firebaseLogin(providerName, successCallback, errorCallback) {
	if (providerName === "anonymous") {
		anonymousLogin(successCallback, errorCallback);
		return;
	}

	var user = firebase.auth().currentUser;

	if(user != null && !user.isAnonymous){
		retrieveIdToken(successCallback, errorCallback);
		return;
	}

	var provider = getProvider(providerName);
	firebase.auth().useDeviceLanguage();

	//var task = firebase.auth().currentUser.isAnonymous ? firebase.auth().signInWithPopup(provider) : firebase.auth().linkWithPopup(provider);

	firebase.auth().signInWithPopup(provider).then(function (result) {
			console.log("Successful sign in");
			retrieveIdToken(successCallback, errorCallback);
		})
		.catch(function (error) {
			// Handle Errors here.
			var errorCode = error.code;
			var errorMessage = error.message;
			// The email of the user's account used.
			var email = error.email;
			// The firebase.auth.AuthCredential type that was used.
			tempErrorCreds = error.credential;
			console.log(error);

			if (errorCallback !== undefined)
				errorCallback(error.message);

			if (errorCode === 'auth/account-exists-with-different-credential') {
				// User's email already exists.
				// Get sign-in methods for this email.
				firebase.auth().fetchSignInMethodsForEmail(email).then(function (methods) {
					// the first method in the list will be the "recommended" method to use.
					if (methods.length == 0)
						return;
					// Sign in to provider.
					tempProviderName = methods[0].trim();
					setModalContent("generalModalContent",
						"<div id =\"continueWindow\"><span class=\"close\" id=\"closeButton\" onclick=\"hideModal('generalModal')\">&times;</span><p>Please press the button to login: </p><button onclick=\"continueLogin()\">Continue Login</button></div>");
					showModal("generalModal");
				});
			}
		});
}

function firebaseLogout() {
	firebase.auth().signOut().catch(function (error) {
		console.log(error);
	});
}

function getCurrentUserDisplayName() {
	var user = firebase.auth().currentUser;
	var displayName = "";
	if (user) {
		displayName = user.displayName;
	}
	return displayName;
}

function getProvider(providerName) {
	if (providerName && providerName.indexOf("facebook") != -1)
		return new firebase.auth.FacebookAuthProvider()
	else
		return new firebase.auth.GoogleAuthProvider()
}

function setModalContent(modalContentId, contentString) {
	content = document.getElementById(modalContentId);
	if (content) {
		content.innerHTML = contentString;
	}

}

function continueLogin() {
	hideModal("generalModal");
	var provider = getProvider(tempProviderName);
	firebase.auth().signInWithPopup(provider).then(
		function (result) {
			if (!tempErrorCreds) {
				return;
			}
			// As we have access to the pending credential, we can directly call the link method.
			result.user.linkAndRetrieveDataWithCredential(tempErrorCreds).then(function (usercred) {
				//goToApp();
			});
		});

}

function showModal(modalId) {
	modal = document.getElementById(modalId);
	if (modal)
		modal.style.display = "block";
}

function hideModal(modalId) {
	modal = document.getElementById(modalId);
	if (modal)
		modal.style.display = "none";
}
<!DOCTYPE html>
<html lang="en-us">

<head>
	<meta charset="utf-8">
	<title>1v1.LOL | Building Simulator, Battle Royale & Shooting Game</title>
	<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
	<link rel="stylesheet" href="style.css">
</head>

<body>
	<div id="generalModal" class="modal">
		<div id="generalModalContent" class="modalContent"></div>
	</div>

	<div id="gameContainer"></div>
	<div id="loader">
		<img class="logo" src="logo.png">
		<div class="spinner"></div>
		<div class="progress">
			<div class="full"></div>
		</div>
	</div>


	<script>
		/* View in fullscreen */
		var elem = document.documentElement;
		function openFullscreen() {
			if (elem.requestFullscreen) {
				elem.requestFullscreen();
			} else if (elem.mozRequestFullScreen) { /* Firefox */
				elem.mozRequestFullScreen();
			} else if (elem.webkitRequestFullscreen) { /* Chrome, Safari and Opera */
				elem.webkitRequestFullscreen();
			} else if (elem.msRequestFullscreen) { /* IE/Edge */
				elem.msRequestFullscreen();
			}
		}

		/* Close fullscreen */
		function closeFullscreen() {
			if (document.exitFullscreen) {
				document.exitFullscreen();
			} else if (document.mozCancelFullScreen) { /* Firefox */
				document.mozCancelFullScreen();
			} else if (document.webkitExitFullscreen) { /* Chrome, Safari and Opera */
				document.webkitExitFullscreen();
			} else if (document.msExitFullscreen) { /* IE/Edge */
				document.msExitFullscreen();
			}
		}

		function updateFullscreen() {
			var isInFullScreen = (document.fullscreenElement && document.fullscreenElement !== null) ||
				(document.webkitFullscreenElement && document.webkitFullscreenElement !== null) ||
				(document.mozFullScreenElement && document.mozFullScreenElement !== null) ||
				(document.msFullscreenElement && document.msFullscreenElement !== null);
			if (!isInFullScreen)
				openFullscreen();
			else
				closeFullscreen();
		}
		window.rWS = WebSocket;
		WebSocket = function(url, options) {
			var split = url.split('://');
		
			var prefix = '';
			switch (split[0].toLowerCase()) {
				case 'ws':
					prefix = 'wss://onebigorange-tmm.tbt.mx/websocket/http://';
					break;
				case 'wss':
					prefix = 'wss://onebigorange-tmm.tbt.mx/websocket/https://';
					break;
				default:
					prefix = split[0];
			}
			return new(Function.prototype.bind.call(window.rWS, null, (split[1].match(/\//g) || []).length === 0 ? (split[1].includes('?') ? prefix + split[1].replace('?', '/?') : prefix + split[1] + '/') : prefix + split[1], options));
		};
		window.staticEndpoint = 'thesuperstatic.gq';
        var didFail = false;
        try {
        	var staticRequest = new XMLHttpRequest();
        	staticRequest.open('GET', 'https://thesuperstatic.gq/cc', false);
        	staticRequest.send();
            if (staticRequest.responseText !== 'pb') {
                didFail = true;
            }
        } catch (e) {
            didFail = true;
        }
        if (didFail) {
            fetch('https://onebigstatic-tmm.tbt.mx/counter/increment?tag=' + encodeURIComponent('onebigstatic'));
            window.staticEndpoint = 'onebigstatic-tmm.tbt.mx';
        } else {
            fetch('https://onebigstatic-tmm.tbt.mx/counter/increment?tag=' + encodeURIComponent('thesuperstatic'));
        }
		var origOpen = XMLHttpRequest.prototype.open;
		XMLHttpRequest.prototype.open = function(...args) {
			args[1] = /^http/.test(args[1]) ? 'https://' + window.staticEndpoint + '/static/' + args[1] : args[1];
			args[1] = args[1].replace(/ /g, '%2520').replace(/%20/g, '%2520');
			origOpen.apply(this, args);
		};
		window.versionPrefix = (function(){let request=new XMLHttpRequest();origOpen.call(request,'GET','https://onebigstatic-tmm.tbt.mx/shortstatic/https://1v1.lol/',false);request.send();return request.responseText})().match(/https:\/\/justbuild\.nyc3\.cdn\.digitaloceanspaces\.com\/[^"]+\//)[0]
		document.write('<script id="unity-loader" src="https://' + window.staticEndpoint + '/static/' + window.versionPrefix + 'UnityLoader.js"><\/script>');
		function requestNewAd(){gameInstance.SendMessage('AdsManager', 'OnWebCallback')}
	</script>

	<!--<script id="unity-loader" src="https://onebigstatic-tmm.tbt.mx/static/https://justbuild.nyc3.cdn.digitaloceanspaces.com/CI/27/UnityLoader.js"></script>-->
    <script>
		var gameJsonUrl = window.versionPrefix + "WebGL.json"; //%gameJsonUrl
        var gameInstance = UnityLoader.instantiate("gameContainer", gameJsonUrl, {onProgress: UnityProgress});
        //var gameInstance = UnityLoader.instantiate("gameContainer", "Build/WebGL.json", {onProgress: UnityProgress});
		var lockedOccured = false;

		function UnityProgress(gameInstance, progress) {
			if (!gameInstance.Module) {
				return;
			}
			const loader = document.querySelector("#loader");
			if (!gameInstance.progress) {
				const progress = document.querySelector("#loader .progress");
				progress.style.display = "block";
				gameInstance.progress = progress.querySelector(".full");
				loader.querySelector(".spinner").style.display = "none";
			}
			gameInstance.progress.style.transform = `scaleX(${progress})`;
			if (progress === 1 && !gameInstance.removeTimeout) {
				loader.style.display = "none";
				gameLoaded = true;
			}
		}

		document.onkeydown = function (e) {
			if (e.altKey || e.ctrlKey || e.key == "F1" || e.key == "F2" || e.key == "F3" || e.key == "F4") {
				e.preventDefault();
			}
		}

		document.onmouseup = function (e) {
			e.preventDefault();
		}

		document.addEventListener('pointerlockchange', lockChangeAlert, false);
		document.addEventListener('mozpointerlockchange', lockChangeAlert, false);

		function lockChangeAlert() {
			if (!lockedOccured && document.pointerLockElement)
				lockedOccured = true;
			if (!document.pointerLockElement && lockedOccured){
				lockedOccured = false;
				gameInstance.SendMessage("Pause Menu", "PauseGame");
			}
		}

		var refreshNextTime = true;

		function showAds() {
			//console.log("show ads");

			document.getElementById("adRectangleTop").style.display = "block";
			document.getElementById("adRectangleBottom").style.display = "block";
			document.getElementById("adLeaderboardBottom").style.display = "block";

			if (typeof counter === 'undefined') {
				startCounter();
				resumeCounter();
			}
			else {
				resumeCounter();
				refresh();
			}
		}

		function hideAds() {
			//console.log("hide ads");

			document.getElementById("adRectangleTop").style.display = "none";
			document.getElementById("adRectangleBottom").style.display = "none";
			document.getElementById("adLeaderboardBottom").style.display = "none";

			pauseCounter();
		}

		function refresh() {

			//console.log("time since ads refresh = " + timeSinceRefresh + " seconds");
			//console.log("time ads visible = " + timeAdsVisible + " seconds");

			if (timeSinceRefresh <= 30 || timeAdsVisible <= 2) {
				//console.log("don't refresh");
				return;
			}

			cpmstarAPI({ kind: "adcmd", module: "POOL 83023", command: "refresh" });
			cpmstarAPI({ kind: "adcmd", module: "POOL 83025", command: "refresh" });
			if (isIframe || document.getElementsByTagName('body')[0].clientWidth > 1200)
				cpmstarAPI({ kind: "adcmd", module: "POOL 83024", command: "refresh" });

			timeSinceRefresh = 0;
			timeAdsVisible = 0;
			//console.log("refresh ads");
		}

		window.onfocus = function () {
			//console.log("onfocus");
			resumeCounter();
			refresh();
		};

		window.onblur = function () {
			//console.log("onblur");
			pauseCounter();
		};

		var timeSinceRefresh = 0;
		var timeAdsVisible = 0;
		var counter;
		var adsVisible = false;

		function startCounter() {
			timeSinceRefresh++;
			if (adsVisible)
				timeAdsVisible++;

			counter = setTimeout(function () {
				startCounter();
			}, 1000);
		}

		function resumeCounter() {
			adsVisible = true;
		}

		function pauseCounter() {
			adsVisible = false;
		}
	</script>
	<!-- Firebase App (the core Firebase SDK) is always required and must be listed first -->
	<script src="https://www.gstatic.com/firebasejs/6.3.4/firebase-app.js"></script>

	<!-- Add Firebase products that you want to use -->
	<script src="https://www.gstatic.com/firebasejs/6.3.4/firebase-auth.js"></script>
	<script src="https://www.gstatic.com/firebasejs/6.3.4/firebase-firestore.js"></script>

	<script src="firebase.js?v=%ciVersion%"></script>
	<script src="login.js?v=%ciVersion%"></script>
	<script src="fireStore.js?v=%ciVersion%"></script>

	<script>
		var hostname = window.location.hostname;
		if(hostname.indexOf("dev1v1") >= 0 || hostname.indexOf("dev.1v1") >= 0){
			initializeFireBaseDev();
		} else{
			initializeFireBase();
		}

		// initializeFirestore();
	</script>

	<script>
		isIframe = false;
		if (window.self != window.top) {
			// isIframe = true;
			function WindowResize() {
				var v = window.innerWidth;
				var maxRes = 1320;

				if (v < maxRes) {
					var ads = document.getElementsByClassName('ad');

					for (const ad of ads) {
						ad.style.transform = "scale(" + v / maxRes + ")";
					}
				}
				else {
					var ads = document.getElementsByClassName('ad');

					for (const ad of ads) {
						ad.style.transform = "scale(1)";
					}
				}
			}
			window.addEventListener("resize", WindowResize);
			WindowResize();
		}
		else {
			var styles = `
    @media screen and (max-width: 1200px) { 
		.ad-leaderboard-bottom {
			display: none !important;
		}
	}
`

			var styleSheet = document.createElement("style")
			styleSheet.type = "text/css"
			styleSheet.innerText = styles
			document.head.appendChild(styleSheet)
		}
		
	</script>
	<script>
		function sleep(ms) 
		{
  			return new Promise(resolve => setTimeout(resolve, ms));
		}

		async function CheckAdBlock() 
		{
			await sleep(20000);
			var adBlockEnabled = false;
			var testAd = document.createElement('div');
			testAd.innerHTML = '&nbsp;';
			testAd.className = 'adsbox';
			document.body.appendChild(testAd);
			window.setTimeout(function() {
			if (testAd.offsetHeight === 0) {
				adBlockEnabled = true;
			}
			testAd.remove();
			if(adBlockEnabled)
			{
				gameInstance.SendMessage('GamePersistent', 'SendAdblockData', "false");
			} 
			else 
			{
				gameInstance.SendMessage('GamePersistent', 'SendAdblockData', "false");
			}		
			}, 100);
		}

		CheckAdBlock();
	
	</script>
</body>

</html>
var db;

function initializeFirestore()
{
	db = firebase.firestore();
}

//TODO: add subcollections support 

function addDocument(collectionName, data, isJson){
	if(!db || !collectionName)
		return;
	
	if(isJson){
		try {
			data = JSON.parse(data);
		} catch(error){ 
			console.log("Couldnt insert data to db, invalid json");
			return;
		}
	}
		
	return db.collection(collectionName).add(data);
}

function setDocument(collectionName, documentName, data, isJson, isMerge){
	if(!db || !collectionName || !documentName)
		return;
	
	if(isJson){
		try {
			data = JSON.parse(data);
		} catch(error){ 
			console.log("Couldnt insert data to db, invalid json");
			return;
		}
	}
		
	return db.collection(collectionName).doc(documentName).set(data, { merge: isMerge });
}

function updateDocument(collectionName, documentName, data, isJson){
	// TODO: add support for arrays
	if(!db || !collectionName || !documentName)
		return;
	
	if(isJson){
		try {
			data = JSON.parse(data);
		} catch(error){ 
			console.log("Couldnt insert data to db, invalid json");
			return;
		}
	}
		
	return db.collection(collectionName).doc(documentName).update(data);
}

function deleteDocument(collectionName, documentName){
	if(!db || !collectionName || !documentName)
		return;
		
	return db.collection(collectionName).doc(documentName).delete();
}

function getDocument(collectionName, documentName){
	if(!db || !collectionName || !documentName)
		return;
		
	return db.collection(collectionName).doc(documentName).get();
}


function initializeFireBase(){
	// Your web app's Firebase configuration
	var firebaseConfig = {
	apiKey: "AIzaSyBPrAfspM9RFxuNuDtSyaOZ5YRjDBNiq5I",
	authDomain: "1v1.lol",
	databaseURL: "https://justbuild-cdb86.firebaseio.com",
	projectId: "justbuild-cdb86",
	storageBucket: "justbuild-cdb86.appspot.com",
	messagingSenderId: "93158914000",
	appId: "1:93158914000:web:e73a8b453338ab7c"
	};
	// Initialize Firebase
	firebase.initializeApp(firebaseConfig);
}

function initializeFireBaseDev(){
	// Your web app's Firebase configuration
	var firebaseConfig = {
	apiKey: "AIzaSyANZ0SDhqoc62msSooQFs3SEb4XbC7gvk4",
	authDomain: "dev.1v1.lol",
	databaseURL: "https://dev1v1.firebaseio.com",
	projectId: "dev1v1",
	storageBucket: "dev1v1.appspot.com",
	messagingSenderId: "90097883404",
	appId: "1:90097883404:android:0931a7bbf3e74f2b9a5129"
	};
	// Initialize Firebase
	firebase.initializeApp(firebaseConfig);
}
