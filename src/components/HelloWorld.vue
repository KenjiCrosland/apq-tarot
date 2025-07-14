<template>
  <!-- Sidebar controls --------------------------------------------------- -->
  <v-navigation-drawer
    app
    class="pa-4"
    color="#1a0050"
    permanent
    width="220"
  >
    <v-btn block class="mb-3" color="#FFCC00" @click="shuffleDeck">Shuffle</v-btn>
    <v-btn block class="mb-3" color="#FFCC00" @click="dealSpread">Deal 3‑Card Spread</v-btn>
    <v-btn block class="mb-3" color="#FFCC00" @click="resetDeck">Reset Deck</v-btn>
    <v-btn block color="#FFCC00" @click="showDebug = !showDebug">
      {{ showDebug ? 'Hide' : 'Show' }} Debug
    </v-btn>
  </v-navigation-drawer>

  <!-- Main content ------------------------------------------------------- -->
  <v-main>
    <v-container
      class="fill-height py-6"
      style="background: radial-gradient(circle at 50% 0%, #471396 0%, #090040 100%);"
    >
      <!-- Question input -->
      <v-row class="mb-6" justify="center">
        <v-col cols="12" md="6">
          <v-text-field
            v-model="question"
            color="#FFCC00"
            dense
            hide-details
            label="Ask a question for this reading"
            outlined
          />
        </v-col>
      </v-row>

      <!-- Spread -->
      <v-row class="mb-8" justify="center">
        <v-col
          v-for="card in spread"
          :key="card.name + card.reversed"
          cols="auto"
        >
          <TarotCard :card="card" @flipped="onFlip" />
        </v-col>
      </v-row>

      <!-- Question headline under spread -->
      <h2 v-if="question" class="question-header mb-8">{{ question }}</h2>

      <!-- Interpretations -->
      <v-row
        v-if="interpretations.length > 0"
        class="mb-10"
        justify="center"
      >
        <v-col
          v-for="item in interpretations"
          :key="item.card.name"
          cols="12"
          md="4"
        >
          <v-card class="pa-3" color="#1a0050">
            <h4 class="yellow--text mb-2">{{ item.card.name }}</h4>
            <p class="white--text">{{ item.text }}</p>
          </v-card>
        </v-col>
      </v-row>

      <!-- Debug info -->
      <v-divider v-if="showDebug" class="mb-4" />
      <pre v-if="showDebug" class="deck-readout mb-4">{{ deckString }}</pre>

      <v-card
        v-if="showDebug"
        class="statistics-card"
        color="#1a0050"
      >
        <v-card-text>
          <h3 class="mb-3">Shuffle Statistics</h3>
          <v-row>
            <v-col cols="12" md="6">
              <div class="stat-group">
                <h4>Card Distribution</h4>
                <div>Total: {{ statistics.total }} <span class="ideal">(78)</span></div>
                <div>Reversed: {{ statistics.reversed }} ({{ statistics.reversalRate }}%)</div>
                <div>Upright: {{ statistics.total - statistics.reversed }} ({{ (100 - statistics.reversalRate).toFixed(1) }}%)</div>
              </div>
            </v-col>
            <v-col cols="12" md="6">
              <div class="stat-group">
                <h4>Orientation Runs</h4>
                <div>Runs ≥3: {{ statistics.runCount }}</div>
                <div>Longest reversed: {{ statistics.longestReversedRun }}</div>
                <div>Longest upright: {{ statistics.longestUprightRun }}</div>
              </div>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
    </v-container>
  </v-main>
</template>

<script setup>
  import { computed, ref } from 'vue'
  import TarotCard from '@/components/TarotCard.vue'

  /* --- reactive state ---------------------------------------------------- */
  const question = ref('')
  const interpretations = ref([])
  const showDebug = ref(false)

  const deck = ref([])
  const spread = ref([])
  const discard = ref([])

  /* --- full 78‑card list ------------------------------------------------- */
  const defaultDeckNames = [
    'The Fool', 'The Magician', 'The High Priestess', 'The Empress', 'The Emperor',
    'The Hierophant', 'The Lovers', 'The Chariot', 'Strength', 'The Hermit',
    'Wheel of Fortune', 'Justice', 'The Hanged Man', 'Death', 'Temperance',
    'The Devil', 'The Tower', 'The Star', 'The Moon', 'The Sun', 'Judgement', 'The World',
    'Ace of Wands', 'Two of Wands', 'Three of Wands', 'Four of Wands', 'Five of Wands',
    'Six of Wands', 'Seven of Wands', 'Eight of Wands', 'Nine of Wands', 'Ten of Wands',
    'Page of Wands', 'Knight of Wands', 'Queen of Wands', 'King of Wands',
    'Ace of Cups', 'Two of Cups', 'Three of Cups', 'Four of Cups', 'Five of Cups',
    'Six of Cups', 'Seven of Cups', 'Eight of Cups', 'Nine of Cups', 'Ten of Cups',
    'Page of Cups', 'Knight of Cups', 'Queen of Cups', 'King of Cups',
    'Ace of Swords', 'Two of Swords', 'Three of Swords', 'Four of Swords', 'Five of Swords',
    'Six of Swords', 'Seven of Swords', 'Eight of Swords', 'Nine of Swords', 'Ten of Swords',
    'Page of Swords', 'Knight of Swords', 'Queen of Swords', 'King of Swords',
    'Ace of Pentacles', 'Two of Pentacles', 'Three of Pentacles', 'Four of Pentacles',
    'Five of Pentacles', 'Six of Pentacles', 'Seven of Pentacles', 'Eight of Pentacles',
    'Nine of Pentacles', 'Ten of Pentacles', 'Page of Pentacles', 'Knight of Pentacles',
    'Queen of Pentacles', 'King of Pentacles',
  ]

  /* --- shuffle helpers --------------------------------------------------- */
  const options = { sideStickiness: 0.5, cutJitter: 15 }

  function riffle (arr, o = options) {
    const cut = Math.floor(arr.length / 2 + (Math.random() * o.cutJitter * 2 - o.cutJitter))
    const left = arr.slice(0, cut)
    const right = arr.slice(cut)
    const merged = []
    let takeLeft = Math.random() < 0.5
    while (left.length > 0 || right.length > 0) {
      if (Math.random() > o.sideStickiness) takeLeft = !takeLeft
      const src = takeLeft ? left : right
      const packet = Math.random() < 0.55 ? 1 : (Math.random() < 0.85 ? 2 : 3)
      for (let i = 0; i < packet && src.length > 0; i++) merged.push(src.shift())
    }
    arr.splice(0, arr.length, ...merged)
  }

  function stripShuffle (arr) {
    const strips = []
    const count = 4 + Math.floor(Math.random() * 4)
    const base = Math.floor(arr.length / count)
    const remaining = [...arr]
    for (let i = 0; i < count - 1; i++) {
      const size = Math.max(5, Math.min(base + Math.floor(Math.random() * 10 - 5), remaining.length - 5))
      strips.push(remaining.splice(0, size))
    }
    strips.push(remaining)
    arr.length = 0
    while (strips.length > 0) {
      const idx = Math.floor(Math.random() * strips.length)
      const s = strips.splice(idx, 1)[0]
      if (Math.random() < 0.1) {
        s.reverse()
        for (const c of s) c.reversed = !c.reversed
      }
      arr.push(...s)
    }
  }

  /* --- persistence ------------------------------------------------------- */
  function saveDeck () {
    const state = { deck: deck.value, spread: spread.value, discard: discard.value }
    localStorage.setItem('tarotDeck', JSON.stringify(state))
  }

  function loadDeck () {
    const saved = localStorage.getItem('tarotDeck')
    if (!saved) return false
    try {
      const s = JSON.parse(saved)
      deck.value = s.deck || []
      spread.value = s.spread || []
      discard.value = s.discard || []
      return true
    } catch {
      return false
    }
  }

  /* --- boot -------------------------------------------------------------- */
  function initDeck () {
    if (loadDeck() && deck.value.length + spread.value.length + discard.value.length === 78) return
    deck.value = defaultDeckNames.map(n => ({ name: n, reversed: false }))
    stripShuffle(deck.value)
    for (let i = 0; i < 3; i++) riffle(deck.value)
    saveDeck()
  }

  /* --- user actions ------------------------------------------------------ */
  function shuffleDeck () {
    const returning = [...spread.value.splice(0), ...discard.value.splice(0)]
    for (const c of returning) if (Math.random() < 0.05) c.reversed = !c.reversed
    deck.value.push(...returning)
    if (Math.random() < 0.7) {
      for (const _ of Array.from({ length: 3 })) {
        riffle(deck.value)
      }
    } else {
      riffle(deck.value)
      stripShuffle(deck.value)
      riffle(deck.value)
    }
    if (Math.random() < 0.6) {
      const cut = Math.floor(deck.value.length * (0.25 + Math.random() * 0.5))
      deck.value = [...deck.value.slice(cut), ...deck.value.slice(0, cut)]
    }
    saveDeck()
  }

  function dealSpread () {
    if (spread.value.length > 0) discard.value.push(...spread.value.splice(0))
    interpretations.value = []
    shuffleDeck()
    for (let i = 0; i < 3 && deck.value.length > 0; i++) {
      const card = deck.value.shift()
      card.faceUp = false
      spread.value.push(card)
    }
    saveDeck()
  }

  function resetDeck () {
    if (!window.confirm('Restore deck to original order?')) return
    localStorage.removeItem('tarotDeck')
    deck.value = defaultDeckNames.map(n => ({ name: n, reversed: false }))
    spread.value = []
    discard.value = []
    interpretations.value = []
    saveDeck()
  }

  /* --- flip handler / lorem --------------------------------------------- */
  function lorem () {
    return 'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer nec odio. Praesent libero. Sed cursus ante dapibus diam.'
  }

  function onFlip (card) {
    if (!interpretations.value.some(i => i.card === card)) {
      interpretations.value.push({ card, text: lorem() })
    }
  }

  /* --- computed ---------------------------------------------------------- */
  const deckString = computed(() =>
    [...deck.value, ...spread.value, ...discard.value]
      .map(c => c.name + (c.reversed ? ' (R)' : ''))
      .join(', '),
  )

  const statistics = computed(() => {
    const all = [...deck.value, ...spread.value, ...discard.value]
    const reversed = all.filter(c => c.reversed).length
    const revRate = (reversed / all.length * 100).toFixed(1)
    const runs = []
    let run = { type: null, length: 0 }
    for (const c of all) {
      const t = c.reversed ? 'rev' : 'up'
      if (t === run.type) run.length++
      else {
        if (run.length >= 3) {
          runs.push(run)
        }
        run = { type: t, length: 1 }
      }
    }
    if (run.length >= 3) runs.push(run)
    const longest = typ => runs.filter(r => r.type === typ).reduce((m, r) => Math.max(m, r.length), 0)
    return {
      total: all.length,
      reversed,
      reversalRate: revRate,
      runCount: runs.length,
      longestReversedRun: longest('rev') || 0,
      longestUprightRun: longest('up') || 0,
    }
  })

  initDeck()
</script>

<style scoped>
.v-btn { color: #090040 !important; }

.question-header {
  text-align: center;
  color: #FFCC00;
  font-size: 1.5rem;
}

.deck-readout {
  color: #FFCC00;
  font-family: monospace;
  white-space: pre-wrap;
}

.statistics-card { border: 1px solid #471396; }
.statistics-card h3 { color: #FFCC00; font-size: 1.2rem; }
.stat-group { color: #ddd; font-size: 0.9rem; line-height: 1.5; }

.ideal { color: #888; font-size: 0.85rem; font-style: italic; }
</style>
