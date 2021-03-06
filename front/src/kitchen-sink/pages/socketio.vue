<template>
  <eb-page>
    <eb-navbar large largeTransparent title="Socket IO" eb-back-link="Back"></eb-navbar>
    <f7-messagebar ref="messagebar" placeholder="Message" @submit="onSubmit">
      <f7-icon md="material:send" slot="send-link"></f7-icon>
    </f7-messagebar>
    <f7-messages ref="messages">
      <f7-messages-title>{{new Date()}}</f7-messages-title>
      <f7-message v-for="(item,index) in messagesData" :key="index" :type="item.type" :name="item.author.name" :avatar="item.author.avatar" :first="isFirstMessage(item, index)" :last="isLastMessage(item, index)" :tail="isTailMessage(item, index)">
        <span slot="text" v-if="item.message.content.text" v-text="`${item.message.id || ''}:${item.message.content.text}`"></span>
      </f7-message>
    </f7-messages>
  </eb-page>
</template>
<script>
const _subscribePath = '/test/party/test';
import Vue from 'vue';
export default {
  data() {
    return {
      io: null,
      messagesData: [],
      messageScene: Vue.prototype.$meta.config.scene,
      subscribeId: null,
      messageOffset: -1,
      messageOffsetPending: -1,
      messageOfflineFetching: false,
      messageClass: {
        module: 'test-party',
        messageClassName: 'test',
      },
      messageIdsToRead: {},
    }
  },
  computed: {
    user() {
      return this.$store.state.auth.user.op;
    },
    userAvatar() {
      let avatar = this.user.avatar;
      if (!avatar) {
        const configBase = this.$meta.config.modules['a-base'];
        avatar = configBase.user.avatar.default;
      }
      return this.$meta.util.combineImageUrl(avatar, 48);
    },
    userAuthor() {
      return {
        userId: this.user.id,
        name: this.user.userName,
        avatar: this.userAvatar,
      };
    },
    userSystem() {
      const configBase = this.$meta.config.modules['a-base'];
      const avatar = this.$meta.util.combineImageUrl(configBase.user.avatar.default, 48);
      return {
        userId: 0,
        name: 'System',
        avatar,
      };
    },
  },
  beforeDestroy() {
    if (this.subscribeId) {
      this.io.unsubscribe(this.subscribeId);
      this.subscribeId = null;
    }
  },
  created() {
    const action = {
      actionModule: 'a-socketio',
      actionComponent: 'io',
      name: 'instance',
    };
    this.$meta.util.performAction({ ctx: this, action }).then(io => {
      this.io = io;
      this.subscribeId = this.io.subscribe(
        _subscribePath,
        this.onMessage.bind(this),
        this.onSubscribed.bind(this), { scene: this.messageScene },
      );
    });
  },
  mounted() {
    this.messagebar = this.$refs.messagebar.f7Messagebar;
    this.messages = this.$refs.messages.f7Messages;
  },
  methods: {
    setMessageOffset(offset) {
      if (this.messageOfflineFetching) {
        if (offset > this.messageOffsetPending) this.messageOffsetPending = offset;
        return;
      }
      if (offset > this.messageOffset) {
        this.messageOffset = offset;
        console.log('---current message offset:', this.messageOffset);
      }
    },
    onMessage({ message }) {
      this._pushMessage(message);
      this.setMessageOffset(message.id);
    },
    onSubscribed() {
      if (this.messageOfflineFetching) return;
      this.messageOfflineFetching = true;
      // get offset
      if (this.messageOffset === -1) {
        this.$api.post('/a/socketio/message/offset', {
          messageClass: this.messageClass,
        }).then(data => {
          this.messageOffset = data.offset;
          if (this.messageOffset === -1) {
            this._offlineFetchStop();
          } else {
            this._offlineFetch();
          }
        }).catch(err => {
          this._offlineFetchStop();
        });
      } else {
        this._offlineFetch();
      }
    },
    _offlineFetch() {
      this.$api.post('/a/socketio/message/select', {
        messageClass: this.messageClass,
        options: {
          offset: this.messageOffset,
        },
      }).then(data => {
        // push
        const list = data.list;
        if (list.length > 0) {
          // offset
          this.messageOffset = list[list.length - 1].id;
          for (const message of list) {
            this._pushMessage(message);
          }
        }
        // next
        if (data.finished) {
          this._offlineFetchStop();
        } else {
          this._offlineFetch();
        }
      }).catch(err => {
        this._offlineFetchStop();
      });
    },
    _offlineFetchStop() {
      this.messageOfflineFetching = false;
      this.setMessageOffset(this.messageOffsetPending);
    },
    _pushMessage(message) {
      if (typeof message.content === 'string') {
        message.content = JSON.parse(message.content);
      }
      this.messagesData.push({
        type: message.userIdTo === this.user.id ? 'received' : 'sent',
        message,
        author: message.userIdFrom === this.user.id ? this.userAuthor : this.userSystem,
      });
      this._messageToRead(message);
    },
    _messageToRead(message) {
      if (message.messageRead === 1) return;
      this.messageIdsToRead[message.id] = true;
      this._performRead();
    },
    _performRead: Vue.prototype.$meta.util.debounce(function() {
      this._performRead2();
    }, 300),
    _performRead2() {
      const messageIds = Object.keys(this.messageIdsToRead);
      this.messageIdsToRead = {};
      this.$api.post('/a/socketio/message/setRead', { messageIds }).then(() => {
        // do nothing
      }).catch(() => {
        // save back
        for (const messageId of messageIds) {
          this.messageIdsToRead[messageId] = true;
        }
      });
    },
    isFirstMessage(item, index) {
      const previousItem = this.messagesData[index - 1];
      if (item.isTitle) return false;
      if (!previousItem || previousItem.type !== item.type || previousItem.author.userId !== item.author.userId) return true;
      return false;
    },
    isLastMessage(item, index) {
      const nextItem = this.messagesData[index + 1];
      if (item.isTitle) return false;
      if (!nextItem || nextItem.type !== item.type || nextItem.author.userId !== item.author.userId) return true;
      return false;
    },
    isTailMessage(item, index) {
      return this.isLastMessage(item, index);
    },
    onSubmit(value, clear) {
      // message
      const message = {
        id: null,
        messageType: 1, // text
        messageFilter: 0,
        messageGroup: 0,
        messageScene: this.messageScene,
        userIdTo: 0,
        userIdFrom: this.user.id,
        content: {
          text: value,
        },
        messageRead: 1,
      };
      this._pushMessage(message);
      // clear
      clear();
      // focus
      if (value) {
        this.messagebar.focus();
      }
      // send
      this.$api.post('test/feat/socketio/publish', {
        message,
        options: {
          scene: this.messageScene,
        },
      }).then(data => {
        message.id = data.id;
        this.setMessageOffset(message.id);
      });

    }
  }
}

</script>
