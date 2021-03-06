<template>
    <div>
        <v-container v-if="!loading">

            <v-alert info>
                {{$t('response_time')}}
            </v-alert>
            <br>
            <h4>{{$t('subject')}}: {{ticket.subject}}</h4>

            <v-card class="grey lighten-4">
                <v-card-row actions class="blue">
                    <v-icon class="white--text">face</v-icon>
                    <v-btn flat class="white--text">{{$t('client')}}</v-btn>
                    <!--<v-spacer></v-spacer>-->
                    <v-icon class="white--text">event</v-icon>
                    <v-btn flat class="white--text">{{ticket.created_at | prettyDate}}</v-btn>
                </v-card-row>
                <v-card-text>
                    <div v-html="ticket.message">
                    </div>
                </v-card-text>
            </v-card>

            <br>

            <div v-for="tmsg in ticketMessages">
                <v-card class="grey lighten-4">

                    <v-card-row actions class="green" v-if="tmsg.staff">
                        <v-icon class="white--text">face</v-icon>
                        <v-btn flat class="white--text">{{$t('staff')}}</v-btn>
                        <!--<v-spacer></v-spacer>-->
                        <v-icon class="white--text">event</v-icon>
                        <v-btn flat class="white--text">{{tmsg.created_at | prettyDate}}</v-btn>
                    </v-card-row>

                    <v-card-row actions class="blue" v-if="!tmsg.staff">
                        <v-icon class="white--text">face</v-icon>
                        <v-btn flat class="white--text">{{$t('client')}}</v-btn>
                        <v-icon class="white--text">event</v-icon>
                        <v-btn flat class="white--text">{{tmsg.created_at | prettyDate}}</v-btn>
                    </v-card-row>

                    <v-card-text>
                        <div v-html="tmsg.message">
                        </div>
                    </v-card-text>
                </v-card>
                <br>
            </div>
            <br>
            <textarea v-bind:placeholder="$t('textarea')" v-model="msg"></textarea>
            <p>
                <span v-if="msg.length >= 0 && msg.length <= 1">{{msg.length}}/3000</span>
                <span id="counter-ok" v-if="msg.length <= 3000 && msg.length >= 2">{{msg.length}}/3000</span>
                <span id="counter-slow-down" v-if="msg.length > 3000">{{msg.length}}/3000</span>
            </p>
            <v-btn v-on:click.native="addTicketMessage(msg)" class="btn-flat-focused">{{$t('send')}}</v-btn>
        </v-container>
    </div>
</template>
<style>
    #counter-ok {
        color: green;
    }

    #counter-slow-down {
        color: red;
    }

    textarea {
        width: 100%;
        border-bottom: 1px solid #ddd;
        outline: none;
        height: 5rem;
    }
</style>
<script>
    import moment from 'moment'
    export default {
        data () {
            return {
                msg: ''
            }
        },
        mounted () {
            this.$emit('view', this.meta())
            moment.locale(this.$lang);
            this.$store.commit('setNavbarTitle', this.$t('header_ticket'))
            this.$store.dispatch('checkSession').then((nosession) => {
                if (nosession) {
                    this.$vuetify.toast.create(this.$t('auth_no'), "right")
                    this.$router.push('/login')
                } else {
                    this.loadTicket()
                }
            })
        },
        computed: {
            pagination () {
                return this.$store.state.pagination
            },
            loading () {
                return this.$store.state.loading
            },
            ticket () {
                return this.$store.state.ticket
            },
            ticketMessages () {
                return this.$store.state.ticketMessages
            }
        },
        filters: {
            prettyDate (unixtimestamp) {
                var timestamp = moment.unix(unixtimestamp);
                //DD.MM.YYYY or "L"
                return timestamp.format("H:mm L")
            }
        },
        preFetch () {
            return this.methods.meta()
        },
        methods: {
            meta() {
                return {
                    title: 'Pagination Component | Vuetify.js',
                    h1: 'Pagination',
                    description: 'Pagination component for Vuetify Framework',
                    keywords: 'vuetify, pagination, components'
                }
            },
            loadTicket() {
                this.$store.commit('setLoading')
                this.$store.dispatch('ticketInfo', {
                    'id': this.$route.params.id
                }).then(() => {
                    //after ticket loaded, load messages in this ticket
                    this.$store.commit('setLoaded')
                    this.loadMessages()
                })
            },
            loadMessages() {
                this.$store.dispatch('ticketMessages', {
                    'id': this.$route.params.id
                }).then(() => {
                    this.$store.commit('setNavbarTitle', "Ticket #" + this.$route.params.id)//FIXME set by API not user input
                })
            },
            addTicketMessage(msg) {
                //this.$store.commit('setLoading')

                //check if duplicate of previous message, precaution to prevent nervous users spamming to backend
                //or just some network problem just occurred
                var messages = this.$store.state.ticketMessages
                var len = messages.length
                var lastMsg = messages[len - 1].message
                if (msg == lastMsg) {
                    this.$vuetify.toast.create(this.$t("msg_duplicate"), "right")
                    //show user that message was probably send earlier when some Internet connection errors occurred
                    this.loadMessages()
                    return false
                }

                //too short?
                if (msg.length < 2) {
                    this.$vuetify.toast.create(this.$t("msg_too_short"), "right")
                    return false
                }

                //too long?
                if (msg.length > 3000) {
                    this.$vuetify.toast.create(this.$t("msg_too_long"), "right")
                    return false
                }

                this.$store.dispatch('ticketAddMessage', {
                    'id': this.$route.params.id,
                    'msg': msg
                }).then(() => {
                    this.$store.commit('setLoaded')
                    this.loadMessages()
                    this.msg = ""
                })
            }
        },
        locales: {
            en: {
                subject: "Subject",
                response_time: "Dear customer, reply to your messages may take up to 48h",
                client: "You",
                staff: "lvlup.pro Staff",
                textarea: "Write your message to staff",
                send: "Send",
                msg_too_long: "Message is too long",
                msg_too_short: "Message is too short",
                msg_duplicate: "Message already sent (duplicate)"
            },
            pl: {
                subject: "Temat",
                response_time: "Szanowny kliencie, odpowiedź może zająć do 48h, prosimy o cierpliwość",
                client: "Ty",
                staff: "Obsługa lvlup.pro",
                textarea: "Napisz swoją wiadomość do obsługi",
                send: "Wyślij",
                msg_too_long: "Wiadomość jest za długa",
                msg_too_short: "Wiadomość jest za krótka",
                msg_duplicate: "Wiadomość już wysłana (duplikat)"
            }
        }
    }
</script>
