<template>
  <div id="app" style="-webkit-app-region: drag">
    <div class="action-bar" :class="{ focus }" style="-webkit-app-region: drag">
      <input v-if="showSearch" ref="input" class="action-input" v-model="filter" />
      <button
        class="action-button"
        @click="toggleGreetings()"
        :class="{ active: filters.greeting }"
      >👋</button>
      <button
        class="action-button"
        @click="toggleFilter('follow')"
        :class="{ active: filters.follow }"
      >
        💚
        <span v-if="follows.length" class="count">{{visibleFollows.length}}</span>
      </button>
      <button
        class="action-button"
        @click="toggleFilter('reward')"
        :class="{ active: filters.reward }"
      >
        🎁
        <span v-if="rewards.length" class="count">{{rewards.length}}</span>
      </button>
      <button
        class="action-button"
        @click="toggleFilter('parked')"
        :class="{ active: filters.parked }"
      >
        🅿️
        <span v-if="parked.length" class="count">{{parked.length}}</span>
      </button>
    </div>
    <cg-message-list @ack="ack" @park="park" :focus="focus" :messages="visibleMessages"/>
  </div>
</template>

<script lang="ts">
/* eslint-disable @typescript-eslint/ban-ts-comment */
import Vue from 'vue';
import * as timeago from 'timeago.js';
import '@fortawesome/fontawesome-free/css/fontawesome.min.css';
import '@fortawesome/fontawesome-free/css/brands.min.css';
import 'flag-icon-css/css/flag-icon.min.css';

import CgMessageList from '@/apps/components/CgMessageList.vue';
import followMessages from '@/apps/lib/followMessages';
import { twitchChat, twitchUsers, twitchRewards } from '@/lib/services';
import Message from '@/interfaces/Message';
import User from '@/interfaces/User';

const getRandomFollowMessage = () => (
  followMessages[Math.floor(Math.random() * followMessages.length)]
);

export default Vue.extend({
  components: {
    CgMessageList,
  },
  data: () => ({
    botId: '519135902',
    broadcasterId: '413856795',
    showSearch: false,
    focus: false,
    filter: '',
    messages: [] as Message[],
    follows: [] as Message[],
    filters: {
      greeting: false,
      follow: false,
      reward: false,
      parked: false,
      vox: false,
    },
  }),
  computed: {
    visibleFollows(): Message[] {
      return this.follows.filter((m) => !m.ack).reverse();
    },
    rewards(): Message[] {
      return this.messages.filter((m) => !m.ack && m.type === 'reward').reverse();
    },
    parked(): Message[] {
      return this.messages.filter((m) => !m.ack && m.parked).reverse();
    },
    greetings(): Message[] {
      const regexp = new RegExp(this.filter, 'gi');
      const seenUser = new Set();
      return this.messages.filter(
        (m) => {
          if (seenUser.has(m.username)) return false;
          const showMessage = !m.ack
            && m.message.match(regexp)
            && !m.parked;
          if (showMessage) {
            seenUser.add(m.username);
            return true;
          }
          return false;
        },
      );
    },
    filteredMessages(): Message[] {
      let username: string | RegExp = '';
      let { filter } = this;
      const usernameMatch = new RegExp('(.*)from\\:([a-z]+)\\b', 'i');
      if (filter.includes('from:')) {
        const matches = filter.match(usernameMatch);
        if (matches && matches[2]) {
          [,, username] = matches;
          filter = filter.replace(`from:${username}`, '').trim();
          username = new RegExp(username, 'gi');
        }
      }
      const regexp = new RegExp(filter, 'gi');
      return this.messages.filter(
        (m) => {
          let showMessage = !m.ack
            && !m.parked;
          if (username) {
            showMessage = showMessage && !!m.username.match(username);
          }
          if (filter) {
            showMessage = showMessage && !!m.message.match(regexp);
          }
          return showMessage;
        },
      );
    },
    visibleMessages(): Message[] {
      if (this.filters.follow) return this.visibleFollows;
      if (this.filters.reward) return this.rewards;
      if (this.filters.parked) return this.parked;
      if (this.filters.greeting) return this.greetings;
      return this.filteredMessages;
    },
  },
  methods: {
    async ack(message: Message) {
      if (message.type === 'reward') {
        this.$set(message, 'ack', true);
      } else {
        await twitchChat.patch(message.id, {
          ack: true,
        });
      }
      if (this.filters.parked || (message.type && message.type in this.filters)) {
        if (this.visibleMessages.length === 0) {
          // @ts-ignore
          this.toggleFilter(this.filters.parked ? 'parked' : message.type);
        }
      }
    },
    async park(message: Message) {
      await twitchChat.patch(message.id, {
        parked: true,
      });
    },
    toggleFilter(item: string) {
      // @ts-ignore
      this.filters[item] = !this.filters[item];
      Object
        .keys(this.filters)
        // @ts-ignore
        // eslint-disable-next-line
        .forEach((name) => item !== name ? this.filters[name] = false : '');
      this.filter = '';
      this.showSearch = false;
    },
    toggleGreetings() {
      if (this.filters.greeting) {
        this.toggleFilter('greeting');
        this.filter = '';
        this.showSearch = false;
      } else {
        this.toggleFilter('greeting');
        this.filter = '\\b(hi|hello|hey|morning|afternoon|evening|howdy|codinggHiYo|VoHiYo)\\b';
        this.showSearch = true;
      }
    },
  },
  async created() {
    window.addEventListener('keydown', (event) => {
      if (event.code === 'KeyF' && event.metaKey) {
        this.showSearch = !this.showSearch;
        setTimeout(() => {
          if (this.$refs.input) {
            // @ts-ignore
            this.$refs.input.focus();
          } else {
            this.filter = '';
          }
        }, 200);
      }
    });
    const messageIds = new Set();
    let messages = await twitchChat.find({
      query: {
        created_at: new Date('2020-08-01'),
      },
    });
    messages = messages
      // @ts-ignore
      .sort((a, b) => new Date(b.created_at) - new Date(a.created_at))
      .slice(0, 1000);
    const names = [
      ...new Set(messages.map((message: Message) => message.username)),
    ];
    const users = await twitchUsers.find({
      query: {
        names,
      },
    });
    const usersByName = users.reduce(
      (byName: Map<string, User>, user: User) => {
        byName.set(user.name, user);
        return byName;
      },
      new Map<string, User>(),
    );
    const follows = [] as Message[];
    const allMessages = messages
      .filter((message: Message) => {
        if (messageIds.has(message.id)) return false;
        messageIds.add(message.id);
        message.timeSent = timeago.format(message.created_at);
        const followedUsername = (message.message.match(
          /Thank you for following on Twitch (.*)!/,
        ) || [])[1];
        if (message.username === 'streamlabs' && followedUsername) {
          if (followedUsername) {
            message.type = 'follow';
            message.message = getRandomFollowMessage().replace(
              /\{\{username\}\}/g,
              `<strong>${followedUsername}</strong>`,
            );
            message.color = '#F8C630';
            message.badges_raw = '';
            message.user = {
              name: '',
              display_name: '👋 Follow  👋',
              logo: 'https://i.imgur.com/rD7b0Ki.png',
              follow: true,
            };
            follows.push(message);
          }
        } else {
          if (message.msg_id === 'highlighted-message') {
            message.type = 'highlight';
          }

          message.user = usersByName.get(message.username) || {
            name: 'Not Found',
          };
        }
        return true;
      });

    this.messages = allMessages;
    this.follows = follows;
    twitchRewards.on('created', (data: any) => {
      const { redemption } = data;
      const content = `${redemption.user.display_name || redemption.user.login} has redeemed: ${redemption.reward.title} - ${redemption.reward.prompt}`;

      const message = {
        id: redemption.id,
        message: content,
        content,
        name: '🎉 Reward Redemption 🎉',
        username: '🎉 Reward Redemption 🎉',
        display_name: '🎉 Reward Redemption 🎉',
        badges_raw: '',
        badges: {},
        created_at: new Date(data.timestamp),
        backgroundColor: redemption.reward.background_color,
        type: 'reward',
        user: {
          name: '',
          display_name: '🎉 Reward Redemption 🎉',
          logo: 'https://i.imgur.com/pukCZL7.png',
          follow: true,
          subscription: false,
        },
      };
      this.messages.unshift(message);
    });
    twitchChat.on('removed', (id: string) => {
      this.messages = this.messages.filter((m) => m.id !== id);
    });
    twitchChat.on('patched', (message: any) => {
      const existingMessage = this.messages.find((m) => m.id === message.id);
      if (existingMessage) {
        Object
          .entries(message)
          .forEach(([prop, value]) => this.$set(existingMessage, prop, value));
      }
    });
    twitchChat.on('created', (message: any) => {
      if (message.user_id === this.botId && message.message.includes('Focus mode ended')) {
        this.focus = false;
        return;
      }
      if (message.user_id === this.broadcasterId && message.message.startsWith('!focus')) {
        const command = message.message;
        if (command.startsWith('!focus-start') || command.startsWith('!focus-resume')) {
          this.focus = true;
        } else if (command.startsWith('!focus-pause') || command.startsWith('!focus-end')) {
          this.focus = false;
        }
        return;
      }
      const followedUsername = (message.message.match(
        /Thank you for following on Twitch (.*)!/,
      ) || [])[1];
      if (message.username === 'streamlabs' && followedUsername) {
        if (followedUsername) {
          message.type = 'follow';
          message.message = getRandomFollowMessage().replace(
            /\{\{username\}\}/g,
            followedUsername,
          );
          message.color = '#F8C630';
          message.badges_raw = '';
          message.user = {
            name: '',
            display_name: '👋 Follow  👋',
            logo: 'https://i.imgur.com/rD7b0Ki.png',
            follow: true,
          };
          this.follows.push(message);
        }
      }
      if (message.msg_id === 'highlighted-message') {
        message.type = 'highlight';
      }
      message.timeSent = timeago.format(message.created_at);
      this.messages.unshift(message);
    });
    setInterval(() => {
      if (this.messages.length > 1000) {
        this.messages = this.messages.slice(0, 1000);
      }
      this.messages.forEach((message) => {
        message.timeSent = timeago.format(message.created_at);
      });
    }, 2000);
  },
});
</script>

<style lang="scss">
*::-webkit-scrollbar {
  display: none;
}

#app {
  position: relative;
}

.action-bar {
  position: fixed;
  top: 0.5rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 100;
  width: 94%;
  margin: 0 auto;
  background: #191d32ee;
  padding: 0.5rem;
  border-radius: 10px;
  display: flex;
  justify-content: flex-end;
  align-items: center;

  .action-input {
    border-radius: 5px;
    flex: 1;
    font-size: 1.5rem;
    line-height: 2.8rem;
    width: 30%;
  }

  .action-button {
    outline: none;
    font-size: 2rem;
    margin: 0.25rem;
    font-family: "Fira Sans", sans-serif;
    color: #EDF2F4;
    background: #595F72;
    border-radius: 5px;
    padding: 0.25rem;
    cursor: pointer;
  }
  .action-button.active,
  .action-button:active {
    border: 2px solid black;
    background: #724e91;
  }

  &.focus {
    opacity: 0.3;
  }
}

.messages-item {
  transition: all 500ms ease-in;
}

.messages-move {
  transition: transform 500ms;
}

.messages-enter-active {
  transition: all 500ms ease-in;
}

.messages-leave-active {
  transition: all 200ms ease-in;
}

// .messages-enter,
.messages-leave-to {
  opacity: 0;
  transform: translateY(-50px) translateX(600px);
}

.message {
  margin: 0.5rem 0;
}

.messages {
  margin-top: 15%;
}

.messages.focus {
  opacity: 0.3;
  filter: blur(1px);
  transition: opacity 1s linear;
}

img[src$="#emote"] {
  height: 50px;
}

a,
a:visited {
  color: #e6af2e;
}

p {
  margin: 0;
}
</style>
