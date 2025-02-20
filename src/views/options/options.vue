<template>
  <main>
    <header>
      <div class="container">
        <transition-group name="fade">
          <div class="alert-box" v-for="err in errors" :key="err">
            <alert
              :text="err"
              type="error"
            />
          </div>
        </transition-group>
      </div>
    </header>
    <section class="container">
      <nav>
        <ul>
          <li :class="{active: activePage === 'general'}" @click="activePage = 'general'">
            <span>{{ i18n('general_settings', 'General settings') }}</span>
          </li>
          <li :class="{active: activePage === 'generator'}" @click="activePage = 'generator'">
            <span>{{ i18n('generator_settings', 'Generator settings') }}</span>
          </li>
          <li :class="{active: activePage === 'blacklist'}" @click="activePage = 'blacklist'">
            <span>{{ i18n('blacklist_settings', 'Blacklist settings') }}</span>
          </li>
          <li v-if="prevSettings" :class="{active: activePage === 'prev_settings'}" @click="activePage = 'prev_settings'">
            <span>{{ i18n('previous_settings', 'Previous settings') }}</span>
          </li>
        </ul>
        <div class="actions">
          <primary-button
            v-if="!$store.state.settingsSaved"
            :text="i18n('save_changes', 'Save changes')"
            :pulse="true"
            @click="saveChanges"
          />
        </div>
      </nav>
      <aside>
        <div v-if="activePage === 'general'">
          <h1>{{ i18n('general_settings', 'General settings') }}</h1>
          <p>{{ i18n('general_settings_hint', 'Change the behavior of the switcher to best fit your needs') }}:</p>

          <ul>
            <enable-switcher/>
            <renew-interval/>
            <renew-on-startup/>
            <js-protection/>
            <custom-ua-list/>
            <remote-ua-list/>
          </ul>
        </div>
        <div v-else-if="activePage === 'generator'">
          <h1>{{ i18n('generator_settings', 'Generator settings') }}</h1>
          <p>{{ i18n('generator_settings_hint', 'Here you can change the agent switching behavior') }}:</p>

          <generator-types/>
        </div>
        <div v-else-if="activePage === 'blacklist'">
          <h1>{{ i18n('blacklist_settings', 'Blacklist settings') }}</h1>
          <p>{{
              i18n('blacklist_settings_hint', 'Blacklist mode - switching enabled everywhere, except the defined ' +
                'domains & rules. Whitelist - on the contrary, disabled everywhere except the specified domains & rules')
            }}:</p>

          <ul>
            <whitelist-mode/>
            <blacklist-domains-list/>
            <blacklist-custom-rules-list/>
          </ul>
        </div>
        <div v-else-if="prevSettings && activePage === 'prev_settings'">
          <h1>{{ i18n('previous_settings', 'Previous settings') }}</h1>
          <p>{{
              i18n(
                'previous_settings_hint',
                'These settings were used by you on the previous extension version. Keep it somewhere or remove',
              )
            }}:</p>

          <pre>{{ prevSettings }}</pre>

          <primary-button
            :text="i18n('remove', 'Remove')"
            @click="removePrevSettings"
          />
        </div>
        <div v-else>
          <h1>(╯°□°)╯︵ ┻━┻</h1>
        </div>
      </aside>
    </section>
    <footer>
      <div class="container">
        <footer-block/>
      </div>
    </footer>
  </main>
</template>

<script lang="ts">
import {defineComponent} from 'vue'
import i18n from '../mixins/i18n'
import {Actions} from '../store/actions'

import Toggle from './common/toggle.vue'
import Alert from './common/alert.vue'
import PrimaryButton from './common/primary-button.vue'
import FooterBlock from './extended/footer-block.vue'

import EnableSwitcher from './controls/enable-switcher.vue'
import RenewInterval from './controls/renew-interval.vue'
import RenewOnStartup from './controls/renew-on-startup.vue'
import JSProtection from './controls/js-protection.vue'
import CustomUAList from './controls/custom-ua-list.vue'
import RemoteUAList from './controls/remote-ua-list.vue'
import GeneratorTypes from './controls/generator-types.vue'
import WhitelistMode from './controls/whitelist-mode.vue'
import BlacklistDomainsList from './controls/blacklist-domains-list.vue'
import BlacklistCustomRulesList from './controls/blacklist-custom-rules-list.vue'

const v2_config_key: string = 'extension_settings_v2'

export default defineComponent({
  components: {
    'toggle': Toggle,
    'alert': Alert,
    'primary-button': PrimaryButton,
    'enable-switcher': EnableSwitcher,
    'renew-interval': RenewInterval,
    'renew-on-startup': RenewOnStartup,
    'js-protection': JSProtection,
    'custom-ua-list': CustomUAList,
    'remote-ua-list': RemoteUAList,
    'footer-block': FooterBlock,
    'generator-types': GeneratorTypes,
    'whitelist-mode': WhitelistMode,
    'blacklist-domains-list': BlacklistDomainsList,
    'blacklist-custom-rules-list': BlacklistCustomRulesList,
  },
  mixins: [i18n],
  data(): {
    activePage: 'general' | 'generator' | 'blacklist' | 'prev_settings'
    errors: string[]
    prevSettings: string | undefined
  } {
    return {
      activePage: 'general',
      errors: [],
      prevSettings: undefined as string | undefined, // TODO remove this property after a some time
    }
  },
  methods: {
    addError(err: Error): void {
      const asString = err.toString()

      if (!this.errors.includes(asString)) {
        this.errors.push(asString)

        setTimeout((): void => {
          this.errors = this.errors.filter(s => s !== err.toString())
        }, 5000)
      }
    },

    handleError(err: Error): void {
      this.addError(err)

      console.error(err)
    },

    saveChanges(): void {
      this.$store.dispatch(Actions.SaveSettings).catch(this.handleError)
    },

    removePrevSettings(): void { // TODO remove this method after a some time
      chrome.storage.sync.remove(v2_config_key)

      this.prevSettings = undefined
      this.activePage = 'general'
    },
  },
  created(): void {
    window.addEventListener('beforeunload', (event): void => {
      if (!this.$store.state.settingsSaved) {
        event.returnValue = 'You have unsaved changes!'
      }
    })

    this.$store.dispatch(Actions.LoadSettings).catch(this.handleError)

    try { // TODO remove this block after a some time
      // load the prev extension settings
      chrome.storage.sync.get(v2_config_key, (data) => {
        if (chrome.runtime.lastError) {
          throw new Error(chrome.runtime.lastError.message)
        }
        if (typeof data === 'object' && data.hasOwnProperty(v2_config_key)) {
          this.prevSettings = JSON.stringify(data[v2_config_key], null, 2)
        }
      })
    } catch (e) {
      this.handleError(e as Error)
    }
  },
})
</script>

<style lang="scss" src="../styles/colors.scss"/>
<style lang="scss" src="./styles/main.scss"/>

<style lang="scss" scoped>
$header-height: 80px;
$footer-height: 3.5rem;

main {
  position: relative;
  min-height: 100vh;

  .container {
    max-width: 990px;
    margin: 0 auto;
    padding: 0 1rem;
  }

  header {
    box-sizing: border-box;
    padding-top: .9rem;
    min-height: $header-height;

    .alert-box:not(:last-child) {
      margin-bottom: .2em;
    }
  }

  section {
    display: flex;

    nav {
      flex: 1;
      padding-bottom: $footer-height;

      ul {
        list-style: none;
        padding: 0;
        margin: 0;

        li {
          padding: 1.2em 0;
          cursor: pointer;

          span {
            position: relative;
          }

          &.active {
            span:before {
              content: '';
              position: absolute;
              left: 0;
              right: 0;
              height: .25em;
              background-color: var(--color-ui-on);
              bottom: -0.7em;
            }
          }

          &.active, &:hover {
            -webkit-text-stroke: 0.7px currentColor;
          }
        }
      }

      .actions {
        margin-top: 2em;

        .btn {
          padding-left: 2em;
          padding-right: 2em;
        }
      }
    }

    aside {
      flex: 3.2;
      padding-left: 1rem;
      padding-bottom: $footer-height;
      margin-top: .5rem;

      h1 {
        font-weight: bold;
        font-size: 2em;
        margin: 0;
      }

      ul {
        list-style: none;
        padding: 0;
        margin: 0;
      }
    }
  }

  footer {
    position: absolute;
    bottom: 0;
    width: 100%;
    height: $footer-height;

    display: flex;
    align-items: center;

    background-color: var(--color-bg-light);
  }
}
</style>
