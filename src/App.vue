<template>
  <div id="app" style="max-width: 24rem; padding: 1rem; margin: 1rem auto">
    <img src="logo.svg" style="width: 100%; display: block">
    <h4>Bridge AHPD Onboarding</h4>

    <div class="message-container">
      <div class="ui error message" v-if="error">{{error}}</div>
      <div class="ui info message" v-if="info">{{info}}</div>
    </div>

    <div v-if="screen === 'SignIn'">
      <div class="ui segment">
        <form class="ui form" @submit="signIn">
          <div class="ui field">
            <label>Email</label>
            <input v-model="email" ref="email" placeholder="Email">
            </div>
          <div class="ui field">
            <label>Password</label>
            <input v-model="password" placeholder="Password" type="password">
          </div>
          <div>
            <button class="ui primary button" :class="{loading: loading}" type="submit">Sign In</button>
          </div>
        </form>
      </div>
    </div>

    <div v-if="screen === 'DataEntry'">
      <form class="ui form" @submit="findParticipant">
        <div class="ui field">
          <label>GUID</label>
          <input v-model="guid" ref="guid" placeholder="Participant GUID">
        </div>
        <div class="ui field">
          <label>Phone Number</label>
          <input v-model="phone" placeholder="Phone">
        </div>
        <div>
          <button class="ui primary button" type="submit" :disabled="!guidValid || !phoneValid" 
            :class="{loading: loading}">Add Participant</button>
          <button class="ui button" @click="signOut">Sign Out</button>
        </div>
      </form>
    </div>

    <div v-if="screen === 'AddToExisting'">
      <div class="message-container">
        <div class="ui segment">
          Phone number {{formattedPhone}} is already registered in the study. 
          Do you wish to assign this GUID to that existing participant?
        </div>
      </div>
      <div>
        <button class="ui primary button" type="submit" 
          :class="{loading: loading}" @click="createIdAddToExisting">Add To Existing Participant</button>
        <button class="ui button" @click="showAddToExisting = false">Cancel</button>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import Vue from 'vue';

const SESSION = "session";
const SESSION_TIMEOUT = "sessionTimeout";
const SESSION_PERIOD = 1000*60*60*20; // 20 minutes
const STUDY_ID = 'sage-mpower-2';
const CLINIC_PREFIX = 'AHPD';
const HOST = 'https://ws.sagebridge.org';

function setSessionToken(sessionToken) {
    localStorage.setItem(SESSION, sessionToken);
    localStorage.setItem(SESSION_TIMEOUT, new Date().getTime()+"");
    return sessionToken || '';
}
function clearSessionToken() {
    localStorage.removeItem(SESSION);
    localStorage.removeItem(SESSION_TIMEOUT);
    return '';
}
function restoreSessionToken() {
  let timestamp = parseInt(localStorage.getItem(SESSION_TIMEOUT));
  if ((timestamp + SESSION_PERIOD) <=  new Date().getTime()) {
    clearSessionToken();
    return '';
  }
  return localStorage.getItem(SESSION) || '';
}

let eventHub = new Vue();
axios.interceptors.request.use(
  conf => {
    eventHub.$emit('before-request');
    return conf;
  },
  error => {
    eventHub.$emit('request-error');
    return Promise.reject(error);
  }
);
axios.interceptors.response.use(
  response => {
    eventHub.$emit('after-response');
    return response;
  },
  error => {
    eventHub.$emit('response-error');
    return Promise.reject(error);
  }
);

export default {
  name: 'app',
  data: function() {
    return {
      clinicPrefix: CLINIC_PREFIX,
      email: '',
      password: '',
      error: '',
      info: '',
      sessionToken: '',
      guid: '',
      phone: '',
      formattedPhone: '',
      loading: false,
      showAddToExisting: false
    }
  },
  created: function() {
    this.sessionToken = restoreSessionToken();
    eventHub.$on('before-request', this.showLoading);
    ['request-error','after-response','response-error'].forEach(
      name => eventHub.$on(name,  this.hideLoading));
  },
  mounted: function() {
    this.$nextTick(() => this.$refs.email.focus());
  },
  beforeDestroy: function () {
    eventHub.$off('before-request', this.showLoading);
    ['request-error','after-response','response-error'].forEach(
      name => eventHub.$off(name,  this.hideLoading));
  },  
  computed: {
    guidValid: function() {
      return /[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}/.test(this.guid);
    },
    phoneValid: function() {
      return /[0-9]{10}/.test(this.phone);
    },
    screen: function() {
      if (this.showAddToExisting) {
        return 'AddToExisting';
      } else if (this.sessionToken) {
        return 'DataEntry';
      } else {
        return 'SignIn';
      }
    }
  },
  methods: {
    showLoading() {
      this.error = '';
      this.info = '';
      this.loading = true;
    },
    hideLoading() {
      this.loading = false;
    },
    signIn: function(event) {
      event.preventDefault();
      axios.post(HOST+'/v3/auth/signIn', {
        study: STUDY_ID, 
        email: this.email, 
        password: this.password
      }).then(response => {
        this.email = '';
        this.password = '';
        this.$nextTick(() => this.$refs.guid.focus());
        this.sessionToken = setSessionToken(response.data.sessionToken);
      })
      .catch(this.showError.bind(this));
    },
    findParticipant: function(event) {
      event.preventDefault();
      axios({
        method: 'POST',
        url: HOST+'/v3/participants/search',
        headers: {'Bridge-Session': this.sessionToken},
        data: { phoneFilter: this.phone }
      })
      .then(this.determineParticipantState.bind(this))
      .catch(this.showError.bind(this));
    },
    determineParticipantState: function(response) {
      // Phone exists, external ID exists, != this ID: notify them the phone is taken
      // Phone exists, external ID exists, == this ID: notify them user already registered
      // Phone exists, external ID null: prompt to add to existing account
      // Phone does not exist, create the external ID (no account), then sign up with an account
      if (response.data.items.length) {
        let record = response.data.items[0];
        let externalId = CLINIC_PREFIX+'-'+this.guid;
        this.formattedPhone = record.phone.nationalFormat;
        if (record.externalId && record.externalId === externalId) {
          // participant is already enrolled
          this.error = `This participant is already enrolled with phone number ${this.formattedPhone}.`;
        } else if (record.externalId && record.externalId !== externalId) {
          // phone number is already assigned to another user
          this.error = `This phone number is already assigned to another user: ${record.externalId}.`;
        } else {
          // record exists without an external ID, proceed to prompt about adding it to this account
          this.formattedPhone = record.phone.nationalFormat;
          this.userId = record.id;
          this.showAddToExisting = true;
        }
      } else {
        // No phone, create the ID and then create the account
        this.createIdAndAccount();
      }
    },
    createIdAndAccount: function() {
      return axios({
        method: 'POST',
        url: HOST+'/v3/externalIds',
        headers: {'Bridge-Session': this.sessionToken},
        data: [CLINIC_PREFIX+'-'+this.guid]
      }).then(() => {
        let account = {
          study: STUDY_ID,
          externalId: CLINIC_PREFIX+'-'+this.guid,
          phone: {regionCode: 'US', number: this.phone},
          dataGroups: ['clinical_consent']
        };
        return axios({
          method: 'POST',
          url: HOST+'/v3/participants',
          headers: {'Bridge-Session': this.sessionToken},
          data: account
        }).then(() => {
          this.info = "Participant added";
          this.guid = '';
          this.phone = '';
          this.formattedPhone = '';
          this.$nextTick(() => this.$refs.guid.focus());
        });
      }).catch(this.showError.bind(this));
    },
    createIdAddToExisting: function() {
      return axios({
        method: 'POST',
        url: HOST+'/v3/externalIds',
        headers: {'Bridge-Session': this.sessionToken},
        data: [CLINIC_PREFIX+'-'+this.guid]
      })
      .then(this.getParticipant.bind(this))
      .then(this.addToExisting.bind(this))
      .then(this.finishAddingGUID.bind(this))
      .catch(this.showError.bind(this));
    },
    getParticipant: function() {
      return axios({
        method: 'GET',
        url: HOST+'/v3/participants/'+this.userId,
        headers: {'Bridge-Session': this.sessionToken}
      });
    },
    addToExisting: function(response) {
      let participant = response.data;
      participant.externalId = CLINIC_PREFIX+'-'+this.guid;
      participant.dataGroups.push('clinical_consent');
      return axios({
        method: 'POST',
        url: HOST+'/v3/participants/'+this.userId,
        headers: {'Bridge-Session': this.sessionToken},
        data: participant
      });
    },
    finishAddingGUID: function() {
      this.info = "Participant has been updated with the GUID.";
      this.guid = '';
      this.phone = '';
      this.formattedPhone = '';
      this.showAddToExisting = false;
      this.$nextTick(() => this.$refs.guid.focus());
    },
    signOut: function(event) {
      event.preventDefault();
      this.guid = '';
      this.phone = '';
      axios({
        method: 'POST',
        url: HOST+'/v3/auth/signOut',
        headers: {'Bridge-Session': this.sessionToken}
      }).then(() => {
        this.sessionToken = clearSessionToken();
        this.showAddToExisting = false;
        this.$nextTick(() => this.$refs.email.focus());
      });
    },
    extractMessage: function(e) {
      if (e && e.response && e.response.data && e.response.data.message) {
        return e.response.data.message;
      } else {
        return JSON.stringify(e);
      }
    },
    showError: function(e) {
      if (e && e.response && e.response.status && e.response.status === 401) {
        this.sessionToken = '';
      }
      let message = this.extractMessage(e);
      if (message.includes('Conflict creating an account')) {
        axios({
          method: 'GET',
          url: HOST+'/v3/participants/externalId%3A'+CLINIC_PREFIX+'-'+this.guid,
          headers: {'Bridge-Session': this.sessionToken}
        }).then(response => {
          this.error = `This participant has already been assigned to another telephone number: ${response.data.phone.nationalFormat}.`;
        });
        return;
      }
      this.error = message;
    }
  }
}
</script>

<style>
html {
  font-size: 17px;
}
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
button + button {
  margin-left: 1rem;
}
.field label {
  text-align: left;
  padding-left: 1.1rem;
}
input {
  width: 100%;
}
.message-container {
  margin-bottom: 1rem; 
  text-align: left;
}
</style>
