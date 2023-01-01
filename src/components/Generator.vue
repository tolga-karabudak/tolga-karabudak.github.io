<template>
  <v-container>
    <!-- Errors -->
    <v-snackbar v-model="errorNotification" timeout="5000" color="red darken-4">
      {{error}}
    </v-snackbar>

    <!-- Assignment -->
    <v-row>
      <v-col cols="12">
        <v-dialog transition="dialog-top-transition" max-width="600" v-model="dialog">
          <template v-slot:default>
            <v-card>
              <v-toolbar class="pl-4" color="green darken-4">
              Hello {{person}}!
              </v-toolbar>
              <v-card-text class="px-8 py-6">
                <div class="text-h6">You have been assigned:</div>
                <p class="px-8">
                  <ul>
                    <li v-for="pick in picks" :key="pick">{{pick}}</li>
                  </ul>
                </p>
              </v-card-text>
            </v-card>
          </template>
        </v-dialog>
      </v-col>
    </v-row>

    <!-- Main form -->
    <v-row>
      <v-col cols="12" md="6">
        <h2 class="headline font-weight-bold mb-5">
          Hitatek Hediye Uygulamasına hoşgeldin.
        </h2>
        <p>
          Baykar'ın can sıkıcı değerlendirmelerinden bunaldık ve böyle bir etkinlik yapmaya karar verdik. Katılımcıları YML biçiminde aşağıya yazman gerekli Örn.(Yasin Faruk: /t Omar gibi)
        </p>

        <p>
          Katılımcıları <kbd>YML</kbd> biçiminde aşağıya yazman gerekli Örn.(Yasin Faruk: /t Omar gibi)
          Sonuçlar aşağıda çıkacak
        </p>
        <h4 class="mt-2">
          Örneğin
        </h4>
        <v-card color="indigo darken-4" class="pa-4">
          <code style="white-space: pre-wrap"  >
            {{example}}
          </code>
        </v-card>

        <p class="py-4">
          Mesela burda yasin faruk ile ergün eşleşemeyecek.
        </p>
        <v-card color="indigo darken-4" class="pa-4">
          <code style="white-space: pre-wrap"  >
            {{example2}}
          </code>
        </v-card>
        <p class="py-4">
          Ama biz her türlü eşleşme istiyoruz. O yüzden yukardakini kullanıcam.
        </p>
        <p>
          Bu arada hiç bir veri alınmıyor veri hazır yapılmış bir uygulamanın github linkine anlık dönüşüyor.
        </p>

      </v-col>
      <v-col cols="12" md="6">
        <v-text-field
            v-model="giftCount"
            type="number"
            label="Number of gifts you would like each person to get"></v-text-field>
        <v-textarea clearable clear-icon="mdi-close-circle" label="YML" v-model="yml" rows=12 :placeholder="`#Something like\n${example}`"></v-textarea>
        <v-btn color="green darken-4" @click="generateLinks">
          Oluştur
        </v-btn>
      </v-col>

      <!-- Results -->
      <v-col cols="12" v-if="links">
        <v-divider></v-divider>
        <h2 class="headline font-weight-bold my-5">
          Sonuçlar
        </h2>
        <p>
          Bu sonuçlara bakmadan size yollayacağım<br>
          Spoiler yok. Sunucuya bağlı olmadığım için video çekip göndericem
        </p>
        <ul class="pl-5 my-5">
          <li v-for="(link, person) in links" :key="person">
            {{person}} <code>{{link}}</code>
          </li>
        </ul>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>

import yaml from 'js-yaml'

const RECURSION_LIMIT = 10000

export default {
  name: 'Generator',
  data: () => ({
    // Error notification
    errorNotification: false,
    error: null,
    // Dialog variable
    person: '',
    picks: [],
    dialog: false,
    // Generator variables
    giftCount: 1,
    yml: '',
    links: null,
    example:
`yasin faruk:
  exclude:
    - ergun
ergun:
  exclude:
    - yasin faruk
omar:
marya:
tolga:`,
    example2:
`yasin faruk:
ergun:
omar:
marya:
tolga:`
  }),
  mounted () {
    const url = new URL(window.location.href)
    try {
      this.person = decodeURIComponent(url.searchParams.get('person'))
      this.picks = JSON.parse(decodeURIComponent(escape(atob(decodeURIComponent(url.searchParams.get('picks')))))) // Per https://stackoverflow.com/questions/56647747/how-to-base64-encode-emojis-in-javascript
    } catch (error) {
      // Ignore
    }

    // See if we can show them their picks
    if (this.person && this.picks && this.picks.length) {
      this.dialog = true
    }
  },
  methods: {
    generateLinks () {
      if (!this.yml.length) {
        this.error = 'You need to input some YML'
        this.errorNotification = true
        return
      }

      this.links = {}
      let matches = null
      try {
        matches = this.match(yaml.load(this.yml), this.giftCount)
      } catch (error) {
        this.links = null
        this.error = error
        this.errorNotification = true
      }

      for (const person in matches) {
        this.links[person] = `${window.location.href.split('?')[0]}?person=${encodeURIComponent(person)}&picks=${encodeURIComponent(window.btoa(unescape(encodeURIComponent(JSON.stringify(matches[person])))))}`
      }
    },
    match (rules, giftCount = 1) {
      if (!Number.isInteger(giftCount)) {
        throw new Error('Gift count must be a number')
      }

      for (let r = 0; r < RECURSION_LIMIT; r++) { //  limit
        // Step 1 load all the options into a map
        const picks = {}
        const picked = {}// Acts as a checksum
        for (const person in rules) {
          picks[person] = []
          picked[person] = 0
        }

        // Step 2 assign people and remove pics from list
        for (let i = 0; i < giftCount; i++) {
          const alreadyPicked = []

          for (const personWhoIsPicking in rules) {
            const options = []
            for (const personWhoIsBeingPicked in picks) { // Make a list of eligible gift people
              if (picks[personWhoIsBeingPicked].length > giftCount ||
                personWhoIsBeingPicked === personWhoIsPicking || // Cannot pick yourself
                (picks[personWhoIsPicking].includes(personWhoIsBeingPicked)) || // Cannot get the same person twice
                (alreadyPicked.includes(personWhoIsBeingPicked)) || // Cannot pick an already picked person this round
                (rules[personWhoIsPicking] && rules[personWhoIsPicking].exclude && rules[personWhoIsPicking].exclude.length && rules[personWhoIsPicking].exclude.includes(personWhoIsBeingPicked))// Cannot be a excluded person
              ) {
                continue // Go to the next person
              }
              options.push(personWhoIsBeingPicked)
            }

            const pick = options[Math.floor(Math.random() * options.length)]// From https://stackoverflow.com/questions/5915096/get-a-random-item-from-a-javascript-array
            if (!pick) {
              continue // null case
            }
            alreadyPicked.push(pick)
            picks[personWhoIsPicking].push(pick)
            picked[pick]++
          }
        }

        // Test
        let passed = true
        for (const person in picked) {
          if (picked[person] !== giftCount) { // Test to make sure everyone got the same amount
            passed = false
          }
        }

        if (passed) {
          console.log(`Matches:\n${yaml.dump(picks)}`)
          console.log(`Counts:\n${yaml.dump(picked)}`)

          return picks
        } else {
          continue // Throw the computational kitchen sink at it
        }
      }

      throw new Error(`Failed to come up with a solution after ${RECURSION_LIMIT} attempts.`)
    }
  }
}
</script>

<style>
/* Fade everything */
.v-application * {  /* Just on application prevents blink. Prevent a white blink 2/2*/
  animation: fadein .5s;
}

@keyframes fadein {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}
</style>
