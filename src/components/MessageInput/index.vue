<template>
  <div class="channel-input"
       :class="{editing:!!message, inThread:!!replyTo}">

    <div class="wrap">

      <text-input
        :key="textInputKey"
        @editLastMessage="$emit('editLastMessage', $event)"
        @cancel="$emit('cancel', $event)"
        @submit="onSubmit"
        @change="onChange"
        :value="(message || {}).message || ''"
        :channels="channelSuggestions"
        :users="userSuggestions"
        class="message-input"
        :class="{'no-files': !showFileUpload}" />

      <button
        v-if="showFileUpload"
        class="upload-button" @click="onPromptFilePicker">

        <span>+</span>
      </button>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import _ from 'lodash'
import TextInput from './TextInput'

export default {
  props: {
    hideFileUpload: false,

    // Message we're editing, if any...
    message: Object,

    // Replying to a message, if any...
    replyTo: Object,

    // Channel we're posting to
    channel: Object,
  },

  data () {
    return {
      inputBoxValue: '',
      cursorIndex: -1,
      value: '',

      // This handles input component redrawing/remounting
      // we need to do it to get because setting input.value to ''
      // does not work as expected (value does not change)
      textInputKey: 0,
    }
  },

  computed: {
    ...mapGetters({
      users: 'users/list',
      channels: 'channels/list',
    }),

    channelID () {
      // Returns channelID from one of the provided params
      return (this.channel || {}).ID || (this.message || {}).channelID || (this.replyTo || {}).channelID
    },

    channelSuggestions () {
      return this.channels.map(c => { return { id: c.ID, value: c.name || c.ID || '' } })
    },

    userSuggestions () {
      return this.users.map(u => { return { id: u.ID, value: u.name || u.ID || '' } })
    },

    showFileUpload () {
      // Hide file upload when editing
      return !this.hideFileUpload || !this.message
    },
  },

  beforeMount () {
    if (!this.channelID) {
      console.error('Could not mount message input without at least one of channel/message/replyTo props')
      return false
    }
  },

  methods: {
    ...mapActions({
      setChannelUnreadCount: 'unread/setChannel',
    }),

    onPromptFilePicker () {
      this.$emit('promptFilePicker', {})
    },

    // Override original submit event and extend event
    // data with submitMeta data.
    onSubmit ({ plain, delta, markdown }) {
      const value = markdown.trim()

      const stdResponse = () => {
        // Trigger remounting
        this.textInputKey++

        // Tell parent we're done with editing.
        this.$emit('cancel', {})
      }

      if (this.message && value.length === 0) {
        if (confirm('Delete this message?')) {
          // @todo a more slick, inline confirmation...
          this.$rest.deleteMessage(this.message.channelID, this.message.ID).then(stdResponse)
        }
      } else if (markdown.length === 0) {
        // nothing to do here...
        return false
      } else if (this.message) {
        // Doing update
        this.$rest.updateMessage(this.message.channelID, this.message.ID, value).then(stdResponse)
      } else if (this.replyTo) {
        // Sending reply
        this.$rest.sendReply(this.replyTo.channelID, this.replyTo.ID, value).then(stdResponse)
      } else if (this.channel) {
        // Sending message
        this.$rest.sendMessage(this.channel.ID, value).then(stdResponse)
        this.setChannelUnreadCount({ ID: this.channel.ID, count: 0, lastMessageID: 0 })
      }
    },

    // Update channel activity once in a while while typing
    onChange: _.throttle(function ({ value }) {
      if (value.length > 1) {
        this.$ws.send({ channelActivity: { ID: this.channelID, kind: 'typing' } })
      }
    }, 2000),
  },

  components: {
    TextInput,
  },
}
</script>

<style lang="scss" scoped>
// @todo remove div.wrap, merge it with base tag
// @todo move all styling and basic positioning inside <text-input>

  @import '@/assets/sass/_0.commons.scss';
  // This component probably won't be used elsewhere,
  // should it be the cas easy to externalize
  .channel-input
  {
    min-height:50px;
    width:100%;
    border:solid 5px $appwhite;
    background-color:$appwhite;
    box-shadow: 0.2rem 0 0.2rem 0 rgba($defaulttextcolor, 0.1);
    .wrap {
      float:left;
      width:100%;
      position:relative;
    }
  }

  .upload-button, .message-input
  {
    border: 1px solid transparent;
    background-color:transparent;
  }

  .upload-button span
  {
    display:inline-block;
    line-height: 1;
  }
  .message-input
  {
    font-family: $crustregular;
    margin-left:50px;
    width:calc(100% - 50px);
    border-radius: 0 5px 5px 0;
    border-left:0;
    float:right;
    font-size:15px;
    padding:0;
  }
  .message-input.no-files
  {
    margin-left: 0;
    width: 100%;
  }

  .upload-button {
    position: absolute;
    height: calc(100%);
    width: 50px;
    color: $appgrey;
    border-radius: 5px 0 0 5px;
    font-size: 1.5rem;
    cursor: pointer;
    text-align: center;
    font-size:30px;
    line-height: 100%;
    float:left;
    z-index: 2;
  }

  .upload-button:focus {
    outline: none;
  }

  .message-input:focus-within {
    outline:none;
    border-color:$appgreen;
    ~ .upload-button
    {
      background-color:rgba($appgreen,0.1);
      border-color:$appgreen;
      color:$appgreen;
    }
  }
  // another background in wide, and no shadow
  @media (min-width: $wideminwidth)
  {
    .channel-input .wrap
    {
      border-radius: 5px;
      border: 1px solid $appgrey;
    }
    .channel-input
    {
      border:solid 15px $appwhite;
      border-top-width:0;
      border-bottom:none;
      padding-bottom:25px;
      border-color: $mainbgcolor;
      background-color:$mainbgcolor;
      box-shadow: none;
    }
  }
</style>