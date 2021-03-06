<template>
  <v-app id="lex-app" toolbar>
    <router-view></router-view>
  </v-app>
</template>

<script>
/*
Copyright 2017-2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Amazon Software License (the "License"). You may not use this file
except in compliance with the License. A copy of the License is located at

http://aws.amazon.com/asl/

or in the "license" file accompanying this file. This file is distributed on an "AS IS"
BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the
License for the specific language governing permissions and limitations under the License.
*/

/* eslint no-console: ["error", { allow: ["warn", "error", "info"] }] */
import Vue from 'vue';
import Vuetify from 'vuetify';

import store from './store';

Vue.use(Vuetify);

export default {
  name: 'lex-app',
  store,
  beforeMount() {
    if (!this.$route.query.embed) {
      console.info('running in standalone mode');
      this.$store.commit('setIsRunningEmbedded', false);
      this.$store.commit('setAwsCredsProvider', 'cognito');
    } else {
      console.info('running in embedded mode from URL: ', location.href);
      console.info('referrer (possible parent) URL: ', document.referrer);
      console.info('config parentOrigin:',
        this.$store.state.config.ui.parentOrigin,
      );
      if (!document.referrer
        .startsWith(this.$store.state.config.ui.parentOrigin)
      ) {
        console.warn('referrer origin: [%s] does not match configured parent origin: [%s]',
          document.referrer, this.$store.state.config.ui.parentOrigin,
        );
      }

      window.addEventListener('message', this.messageHandler, false);
      this.$store.commit('setIsRunningEmbedded', true);
      this.$store.commit('setAwsCredsProvider', 'parentWindow');
    }

    this.$store.commit('setUrlQueryParams', this.$route.query);
  },
  mounted() {
    Promise.all([
      this.$store.dispatch('initCredentials'),
      this.$store.dispatch('initRecorder'),
      this.$store.dispatch('initBotAudio'),
      this.$store.dispatch('getConfigFromParent')
        .then(config => this.$store.dispatch('initConfig', config)),
    ])
      .then(() =>
        Promise.all([
          this.$store.dispatch('initMessageList'),
          this.$store.dispatch('initPollyClient'),
          this.$store.dispatch('initLexClient'),
        ]),
      )
      .then(() => (
        (this.$store.state.isRunningEmbedded) ?
          this.$store.dispatch('sendMessageToParentWindow',
            { event: 'ready' },
          ) :
          Promise.resolve()
      ))
      .catch((error) => {
        console.error('could not initialize application while mounting:', error);
      });
  },
  methods: {
    // messages from parent
    messageHandler(evt) {
      // security check
      if (evt.origin !== this.$store.state.config.ui.parentOrigin) {
        console.warn('ignoring event - invalid origin:', evt.origin);
        return;
      }
      if (!evt.ports) {
        console.warn('postMessage not sent over MessageChannel', evt);
        return;
      }
      switch (evt.data.event) {
        case 'ping':
          console.info('pong - ping received from parent');
          evt.ports[0].postMessage({
            event: 'resolve',
            type: evt.data.event,
          });
          break;
        // received when the parent page has loaded the iframe
        case 'parentReady':
          evt.ports[0].postMessage({ event: 'resolve', type: evt.data.event });
          break;
        case 'toggleMinimizeUi':
          this.$store.dispatch('toggleIsUiMinimized')
            .then(() => {
              evt.ports[0].postMessage(
                { event: 'resolve', type: evt.data.event },
              );
            });
          break;
        case 'postText':
          if (!evt.data.message) {
            evt.ports[0].postMessage({
              event: 'reject',
              type: evt.data.event,
              error: 'missing message field',
            });
            return;
          }

          this.$store.dispatch('postTextMessage',
            { type: 'human', text: evt.data.message },
          )
            .then(() => {
              evt.ports[0].postMessage(
                { event: 'resolve', type: evt.data.event },
              );
            });
          break;
        default:
          console.warn('unknown message in messageHanlder', evt);
          break;
      }
    },
  },
};
</script>

<style>
@import '../node_modules/roboto-fontface/css/roboto/roboto-fontface.css';
@import '../node_modules/material-design-icons/iconfont/material-icons.css';
@import '../node_modules/vuetify/dist/vuetify.min.css';
#lex-app {
  display: flex;
  height: 100%;
  width: 100%;
}
body, html {
  overflow-y: hidden;
}
</style>
