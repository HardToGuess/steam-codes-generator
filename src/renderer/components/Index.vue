<template>
  <div id="wrapper">
    <main>
      <div class="row">
        <div class="col s12 text-centered">
          <span class="text-sm as-block width100">
            Important: time on your PC must be configured correctly, otherwise you'll get wrong codes<br/>
            Generating new code every 1 second, every 30 seconds code changes.
            <br/>
            <span v-if="sharedSecret">
              Current shared secret: {{sharedSecret}}
            </span>
            <span v-if="!sharedSecret" class="text-lg"><b>Shared secret is not provided</b></span><br/>
          </span>
          <a href="javascript:;" class="btn  green darken-2 start-generating-codes" v-if="sharedSecret && !generatingCodes" @click="toggleCodeGenerating()">Start generating codes</a>
          <a href="javascript:;" class="btn red start-generating-codes" v-else-if="sharedSecret && generatingCodes" @click="toggleCodeGenerating()">Stop</a>

        </div>

        <div class="container">
          <div>
            <input type="text" placeholder="Copy shared secret to this field" id="sharedSecretInput" v-model="sharedSecretInputText" class="shared-secret-input">
            <a href="javascript:;" class="btn grey darken-4 width100" @click="updateSharedSecret()">Update secret</a>
          </div>
        </div>
      </div>

      <div class="col s12 text-centered">
        <span class="code">{{code}}</span>
        <br/>
        <a href="javascript:;" @click="copyToClipBoard()" v-if="code">{{copyToClipBoardNotice}}</a>
      </div>
      <!-- footer -->
      <div class="footer text-centered">
        <a href="javascript:;" @click="open('https://github.com/HardToGuess/steam-codes-generator')">Github repository</a>
        &nbsp;
        <a href="javascript:;" @click="openConfigDirectory()">Open config directory</a>
      </div>
      <!-- /footer -->
    </main>
  </div>
</template>

<script>
  const remote = require('electron').remote
  const path = require('path')
  const fs = require('fs')
  const SteamTotp = require('steam-totp')
  const { clipboard } = require('electron')
  const {dialog} = require('electron').remote
  const {shell} = require('electron')

  export default {
    name: 'index',
    data () {
      return {
        code: null,
        generatingCodes: false,
        appPath: remote.app.getAppPath(),
        sharedSecret: '',
        sharedSecretInputText: '',
        configDirectoryName: '',
        copyToClipBoardNotice: 'copy to clipboard'
      }
    },
    methods: {
      // Opens link using default user web-browser
      open (link) {
        this.$electron.shell.openExternal(link)
      },
      // Read config file, is file not exists, create new. Returns config object
      readConfigFile () {
        let configFile
        try {
          const readConfigFile = fs.readFileSync(`${this.configDirectoryName}\\config.json`)
          configFile = JSON.parse(readConfigFile)
        } catch (err) {
          if (err.code === 'ENOENT') {
            this.writeConfigFile({ sharedSecret: '' })
          }
          return console.log(err)
        }
        // Update current shared secret
        this.sharedSecret = configFile.sharedSecret
        return configFile
      },
      // Write shared secret to config file
      writeConfigFile ({ sharedSecret }) {
        const config = { sharedSecret }
        try {
          fs.writeFileSync(`${this.configDirectoryName}\\config.json`, JSON.stringify(config))
          this.readConfigFile()
        } catch (err) {
          console.log('Error while creating config file', err)
        }
      },
      async updateSharedSecret () {
        if (!this.sharedSecretInputText) {
          const sharedSecretResetResponse = await this.showSharedSecretResetDialog()
          if (sharedSecretResetResponse === 1) return
        }
        // Write new shared secret to config
        this.writeConfigFile({ sharedSecret: this.sharedSecretInputText })
        // Clear input
        this.clearSharedSecretInput()
      },
      showSharedSecretResetDialog () {
        return new Promise((resolve, reject) => {
          dialog.showMessageBox({
            type: 'info',
            buttons: ['OK', 'Cancel'],
            title: 'Rly?',
            message: 'Clear shared secret?'
          }, (response) => {
            resolve(response)
          })
        })
      },
      clearSharedSecretInput () {
        this.sharedSecretInputText = ''
      },
      toggleCodeGenerating () {
        this.generatingCodes = !this.generatingCodes
        const codeGenerationInterval = setInterval(() => {
          // If user clicked 'stop', clear interval
          if (!this.generatingCodes) {
            return clearInterval(codeGenerationInterval)
          }
          if (!this.sharedSecret) {
            this.code = ''
            return
          }
          // Generate auth code
          this.code = SteamTotp.generateAuthCode(this.sharedSecret)
        }, 1000)
      },
      copyToClipBoard () {
        clipboard.writeText(this.code, 'selection')
        this.copyToClipBoardNotice = 'copied'
        setTimeout(() => { this.copyToClipBoardNotice = 'copy to clipboard' }, 500)
      },
      openConfigDirectory () {
        shell.openItem(path.dirname(this.appPath))
      }
    },
    created () {
      // On startup, get app directory and load config file
      this.configDirectoryName = path.dirname(this.appPath)
      this.readConfigFile()
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Roboto:300,400,500,700');

  body {
    margin-top: 25px;
    font-family: 'Roboto', sans-serif; 
  }

  .text-centered {
    text-align: center !important;
  }

  .square-borders {
    border-radius: 0px !important;
  }

  .text-sm {
    font-size: 0.8em;
  }

  .as-block {
    display: block;
  }

  .width100 {
    min-width: 100% !important;
  }

  .start-generating-codes {
    margin-top: 25px;
    display: inline-block;
  }

  .footer {
     position: fixed;
     left: 0;
     bottom: 0;
     width: 100%;
     background-color: #ededed;
     text-align: center;
  }

  .code {
    display: block;
    width: 100%;
    font-size: 5em;
  }

  .text-lg {
    font-size: 2em;
  }
</style>
