<template>
    <div class="messages_wrap">
        <div v-show="is_loading" class="mercurius__loading">
            <svg class="ic"><use xlink:href="#icon-ani-puff"></use></svg>
        </div>

        <vue-scroll
            ref="wrap"
            :ops="ops"
            @handle-scroll="onScroll"
        >
            <div class="messages">

                <!-- Typing indicator -->
                <div class="mt-2" v-if="is_typing && !!conversationId ">
                    <div class="message_row msg_received">
                        <div class="message">
                            <img
                                class="message__avatar"
                                :alt="conversation.user"
                                :src="conversation.avatar"
                            />
                            <div
                                class="message__body"
                                v-b-tooltip.show.right
                                :title="'is_typing' | __"
                                >
                                <svg class="ic"><use xlink:href="#icon-ani-dots"></use></svg>
                            </div>
                        </div>
                    </div>
                </div>


                <template v-for="(msg, idx) in messages">
                    <div
                        class="message_row"
                        :class="msgClass(msg, idx)"
                        :key="msg.id"
                        ref="msg"
                    >
                        <!-- Messages Datetime -->
                        <div
                            v-if="showDatetime(msg, idx)"
                            class="messages__datetime"
                        >{{msg.created_at | datetimeRow}}</div>


                        <div class="message">
                            <!-- Avatar -->
                            <img
                                v-if="showAvatar(msg, idx)"
                                class="message__avatar"
                                :alt="msg.sender"
                                :src="conversation.avatar"
                            />

                            <!-- Message -->
                            <div class="message__body"
                                v-b-toggle="'aux'+msg.id"
                                v-text="msg.message">
                            </div>

                            <!-- Delete btn -->
                            <button
                                class="message__action btn btn-link"
                                title="Delete"
                                type="button"
                                v-show="msg.id > 0"
                                @click="deleteMessage(msg)"
                            ><svg class="ic ic-bin"><use xlink:href="#icon-bin"></use></svg>
                            </button>
                        </div>


                        <!-- Message Datetime -->
                        <b-collapse
                            class="message__datetime"
                            :id="'aux'+msg.id"
                            nofade
                        >
                            {{msg.created_at | datetimeSingle}}
                            <button
                                v-if="!received(msg)"
                                class="text-secondary btn btn-link p-0 pb-1"
                                v-b-tooltip:message_row.topleft
                                :title="deliveryStatus(msg)"
                            >
                                <svg class="ic ic-clock" v-if="!msg.seen_at"><use xlink:href="#icon-clock"></use></svg>
                                <svg class="ic ic-check" v-else><use xlink:href="#icon-check"></use></svg>
                            </button>
                        </b-collapse>
                    </div>
                </template>
            </div>
        </vue-scroll>
    </div>
</template>

<script>
import MessagesHttp from './messages-http';

export default {
    props: ['conversation'],


    mixins: [
        MessagesHttp,
    ],


    data() {
        return {
            is_typing: false,
            type_timeout: null,
            ops: {
                scrollPanel: {
                    initialScrollY: 100,
                },
                rail: {
                    size: '10px',
                    gutterOfSide: '0',
                },
            },
        }
    },


    created() {
        Bus.$on('mercuriusMessageReceived', (usr, sender, msg) => this.onMessageReceived(sender, msg))
        Bus.$on('mercuriusMessageSent', (msg) => this.onMessageSent(msg))
        Bus.$on('mercuriusConversationOpen', conv => this.onLoadMessages(conv));
        Bus.$on('mercuriusConversationClose', () => this.loadMessagesReset());
        Bus.$on('mercuriusConversationDeleted', user => this.onConversationDeleted(user));
        Bus.$on('mercuriusUserTyping', usr => this.onTyping(usr))
    },


    filters: {
        datetimeSingle(value) {
            return moment.utc(value).local().format('D MMM YYYY HH:mm')
        },

        datetimeRow(value) {
            let m = moment.utc(value).local()
            let h = moment.utc().local().diff(m, 'hours', true)

            if (h < 24) return m.format('HH:mm')            // 14:00
            if (h < 24*7) return m.format('ddd HH:mm')      // Mon 14:00
            if (h < 24*365) return m.format('D MMMM HH:mm') // 25 August 14:00
            return m.format('DD/MM/YYYY HH:mm')             // 25/08/2017 14:00
        },
    },


    methods: {
        // UI helpers
        //
        showDatetime(msg, idx) {
            if (!this._hasMsg(idx+1)) return true;
            return this._sameDay(msg, idx);
        },
        showAvatar(msg, idx) {
            if (!this.received(msg)) return false;
            return !this._sameSender(msg, idx) || this._sameDay(msg, idx);
        },
        msgClass(msg, idx) {
            return (this.received(msg) ? 'msg_received' : 'msg_sent')
                 + (this._sameSender(msg, idx) ? '' : ' new_sender')
        },
        deliveryStatus(msg) {
            return !!msg.seen_at
                ? __('msg_seen_at')+' '+moment.utc(msg.seen_at).local().format('D MMM YYYY HH:mm')
                : __('msg_sent');
        },
        // check if message was received or sent
        received(msg) {
            return msg.sender !== Mercurius.user.slug
        },


        // Private helpers
        //
        _hasMsg(idx) {
            return (!!this.messages[idx]);
        },
        // Check if message was sent on the same day
        _sameDay(msg, idx) {
            return !moment(msg.created_at)
                   .isSame(moment(this.messages[idx+1].created_at), 'day');
        },
        // Check if message was sent by the same user
        _sameSender(msg, idx) {
            if (!this._hasMsg(idx+1)) return false;
            return (this.messages[idx+1].sender === msg.sender);
        },
        _scrollTo(y_val) {
            setTimeout(() => {
                this.$refs.wrap.scrollTo({x: 0, y: y_val}, false)
            }, 250);
        },
        _appendMsg(msg) {
            this.messages.unshift(msg)
            this._scrollTo('100%')
        },


        // Event handlers
        //
        onTyping(usr) {
            if (this.conversationId != usr) return
            if (this.type_timeout) clearTimeout(this.type_timeout);
            this.type_timeout = setTimeout(() => {
                this.is_typing = false;
            }, 2000);
            this.is_typing = true
            this._scrollTo('100%')
        },
        onMessageSent(msg) {
            this._appendMsg(msg)
        },
        onMessageReceived(sender, msg) {
            if (this.conversationId === sender.slug) this._appendMsg(msg)
        },
        onConversationDeleted(user) {
            if (this.conversationId === user) this.loadMessagesReset()
        },
        onLoadMessages(conv) {
            this.loadMessagesStart(conv.slug)
                .then(() => {
                    Bus.$emit('mercuriusConversationLoaded', conv);
                    setTimeout(() => this._scrollTo('100%'), 125);
                });
        },
        onScroll(barY, barX, e) {
            if (barY.scrollTop > 20 || this.offset < 0) return false;

            this.loadMessages()
                .then(() => this._scrollTo('25%'));
        },
    }
};
</script>
