<template>
    <div class="hello">
        <Bubble text class="bubble-outline" v-show="isLoaded">
            <div v-show="isLoggedIn">
                <p>
                    {{
                        $content(
                            'already-logged-in',
                            'You are already logged in, you many continue to vote.'
                        )
                    }}
                </p>

                <p>
                    <button
                        class="button is-large is-centered is-link"
                        @click="$router.push('/chose')"
                    >
                        {{ $ui('vote', 'Vote') }}
                    </button>
                </p>
            </div>

            <div v-show="!isLoggedIn">
                <div class="field">
                    <div class="control">
                        <vue-tel-input
                            class="phone-number-input"
                            v-model="phoneNumber"
                            @onInput="onInput"
                            :placeholder="$ui('phone-number', 'Phone Number')"
                        ></vue-tel-input>
                    </div>
                </div>

                <div class="field is-grouped is-grouped-centered">
                    <div class="control" v-show="isValid">
                        <input
                            id="send-sms-button"
                            class="button is-large is-centered"
                            type="button"
                            :value="$ui('send', 'Send')"
                            @click="onClick"
                        />
                    </div>
                    <div v-show="!isValid" class="is-size-8 is-light is-italic">
                        {{
                            $content(
                                'home-begin',
                                'To begin enter your valid US cell phone number.'
                            )
                        }}
                    </div>
                </div>
            </div>

            <div v-show="!isValid">
                <hr />
                <div class="padded">&nbsp;</div>
                <div class="is-2">
                    <router-link
                        tag="button"
                        to="/manifesto"
                        class="button is-link"
                    >
                        {{ $ui('manifesto-link', 'The Voterify Manifesto') }}
                    </router-link>
                </div>
                <div class="padded">&nbsp;</div>
                <div class="is-2">
                    <router-link
                        tag="button"
                        to="/instructions"
                        class="button is-link"
                    >
                        {{ $ui('instructions-link', 'How to use Voterify') }}
                    </router-link>
                </div>
                <div class="padded">&nbsp;</div>
                <div class="is-2">
                    <router-link
                        tag="button"
                        to="/track"
                        class="button is-link"
                    >
                        {{ $ui('track-link', 'Track your Vote') }}
                    </router-link>
                </div>
                <div class="padded">&nbsp;</div>
                <div v-show="isLoggedIn && isAdmin" class="is-2">
                    <router-link
                        tag="button"
                        to="/admin"
                        class="button is-link"
                    >
                        {{ $ui('admin', 'Admin') }}
                    </router-link>
                </div>
                <hr />
                <div class="is-2">
                    {{ $content('home-footer', 'Voting made whole.') }}
                </div>
            </div>

            <div class="page-counter" v-show="isValid">
                <progress-counter
                    currentPage="0"
                    pageCount="4"
                ></progress-counter>
            </div>
        </Bubble>
    </div>
</template>

<script lang="ts">
import '@/styles/pages/home.scss'
import { EventHub } from '@/factory/event-hub'
import { Bubble, TextInput, ProgressCounter } from '@/components/ui/all'
import { Component, Prop, Vue } from 'vue-property-decorator'
import firebase from '@/factory/firebase-provider'
import { constants } from '@/factory/constants'
import { session } from '@/factory/session'
import firebaseAuth from '@/factory/firebase-auth'
import CookiesModal from '@/components/modals/cookies.vue'
import Phone from '@/mixins/phone'
import Permissions from '@/models/permissions'

import { mapState } from 'vuex'

@Component({
    components: {
        Bubble,
        TextInput,
        ProgressCounter
    },
    mixins: [Phone],
    computed: {
        ...mapState('user', ['permissions', 'isLoggedIn'])
    }
})
export default class HomePage extends Vue {
    @Prop() private msg!: string
    @Prop() private phone!: string
    private permissions!: Permissions
    public isLoggedIn!: Boolean

    public data() {
        return {
            constants: constants,
            phoneNumber: this.phone,
            isLoaded: false,
            isValid: false
        }
    }

    public mounted() {
        this.load()
    }

    public load() {
        const instance = this as any

        if (!instance.areCookiesEnabled()) {
            EventHub.$emit('showModal', {
                modal: CookiesModal,
                callBackFn: instance.onClickModalOk
            })
        }

        ;(window as any).recaptchaVerifier = new firebase.auth.RecaptchaVerifier(
            'send-sms-button',
            {
                size: 'invisible',
                callback(response) {
                    instance.onClick()
                }
            }
        )
        const phoneInput: Element = document.querySelectorAll('[type="tel"]')[0]
        ;(phoneInput as any).focus()
        instance.isLoaded = true

        const user = session.getUser()
        const voter = session.getVoter()
    }

    public onInput({ number, isValid, country }) {
        ;(this as any).isValid = isValid
    }

    public onClick() {
        const instance = this as any
        const phoneNumber = instance.phoneToRecaptcha((this as any).phoneNumber)
        const appVerifier = (window as any).recaptchaVerifier
        return firebaseAuth
            .phone(phoneNumber, appVerifier)
            .then(function () {
                instance.$router.push('/sms')
            })
            .catch(function (error) {
                EventHub.$emit('showPageLoader', {
                    message: error.message,
                    timeout: 2000,
                    callBackFn: instance.onRestartCaptcha
                })
            })
    }

    public onRestartCaptcha() {
        // tslint:disable-next-line
    }

    public areCookiesEnabled() {
        try {
            document.cookie = 'cookietest=1'
            const cookiesEnabled = document.cookie.indexOf('cookietest=') !== -1
            document.cookie =
                'cookietest=1; expires=Thu, 01-Jan-1970 00:00:01 GMT'
            return cookiesEnabled
        } catch (e) {
            return false
        }
    }

    public onClickModalOk() {
        const instance = this as any
        EventHub.$emit('showPageLoader', {
            message: (instance as any).$content(
                'check-for-cookies',
                'Checking for cookie power...'
            ),
            callBackFn: instance.load,
            timeout: 1000
        })
    }

    public get isAdmin() {
        return this.permissions.type === 'admin'
    }
}
</script>
