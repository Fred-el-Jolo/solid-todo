<script setup lang="ts">
 import {
   getSolidDataset,
   getThing,
   getThingAll,
   getStringNoLocale,
   getNamedNodeAll,
   getWellKnownSolid
 } from "@inrupt/solid-client";
 import { Session } from "@inrupt/solid-client-authn-browser";
 import { VCARD, RDF } from "@inrupt/vocab-common-rdf";
import { computed, ref } from "vue";

const solidIdentityProvider = ref("https://solidcommunity.net");

const session = new Session();

const isLogged = ref(session.info.isLoggedIn);

const userName = ref('');

const loggedMessage = computed(() => isLogged.value ? `Logged in as ${userName.value}` : 'Not logged yet');


// 1a. Start Login Process. Call session.login() function.
async function login() {
  if (!session.info.isLoggedIn) {
    await session.login({
      oidcIssuer: solidIdentityProvider.value,
      clientName: "solid-todo",
      redirectUrl: window.location.href
    });
  }
}

// 1b. Login Redirect. Call session.handleIncomingRedirect() function.
// When redirected after login, finish the process by retrieving session information.
async function handleRedirectAfterLogin() {
  await session.handleIncomingRedirect(window.location.href);
  if (session.info.isLoggedIn) {
    isLogged.value = true;

    const webID = session.info.webId;
    // The WebID can contain a hash fragment (e.g. `#me`) to refer to profile data
    // in the profile dataset. If we strip the hash, we get the URL of the full
    // dataset.
    const profileDocumentUrl = new URL(webID);
    profileDocumentUrl.hash = "";

    // To write to a profile, you must be authenticated. That is the role of the fetch
    // parameter in the following call.

    let myProfileDataset = await getSolidDataset(profileDocumentUrl.href, {
      fetch: session.fetch
    });

    // The profile data is a "Thing" in the profile dataset.
    let profile = getThing(myProfileDataset, webID);
  
    // Get the formatted name (fn) using the property identifier "http://www.w3.org/2006/vcard/ns#fn".
    // VCARD.fn object is a convenience object that includes the identifier string "http://www.w3.org/2006/vcard/ns#fn".
    // As an alternative, you can pass in the "http://www.w3.org/2006/vcard/ns#fn" string instead of VCARD.fn.

    const formattedName = getStringNoLocale(profile, VCARD.fn);

    if (formattedName != null) {
      userName.value = formattedName;
    }

    const PRIVATE_FOLDER = 'https://jolo.solidcommunity.net/private/';

    let privateFolder = await getSolidDataset(PRIVATE_FOLDER, {
      fetch: session.fetch
    });

    let privateContents = await getThingAll(privateFolder);

    for (const privateContent of privateContents) {
      if (privateContent.url !== PRIVATE_FOLDER) {
        let freds = await getSolidDataset(`${privateContent.url}`, {
          fetch: session.fetch
        });
        let fredContents = await getThingAll(freds);
        for (const fredContent of fredContents) {
          if (fredContent.url !== privateContent.url) {
            // console.log(fredContent);
            const fredTypes = getNamedNodeAll(fredContent, "http://www.w3.org/1999/02/22-rdf-syntax-ns#type");
            // console.log(fredTypes);

            // let testeee = getPropertyAll(fredContent);
            // console.log(testeee);

            let freds2 = await getSolidDataset(`${fredContent.url}`, {
              fetch: session.fetch
            });

            console.log(freds2);

            //const testeee = getTurtleParser()
          }
        }
      }
    }
  }
}

// The example has the login redirect back to the index.html.
// This calls the function to process login information.
// If the function is called when not part of the login redirect, the function is a no-op.
if (!session.info.isLoggedIn) {
handleRedirectAfterLogin();
}


</script>

<template>
  <p>Solid Identity Provider</p>
  <input type="text" :value="solidIdentityProvider">
  <button @click="login">Login</button>
  <p>{{ loggedMessage }}</p>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
