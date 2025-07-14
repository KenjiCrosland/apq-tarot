<template>
  <div class="tarot-card" @click="flip">
    <div :class="['card-inner', { flipped: !card.faceUp }]">
      <!-- FRONT -->
      <div
        class="card-face front"
        :style="card.faceUp && card.reversed ? 'transform: rotate(180deg);' : ''"
      >
        {{ card.name }}
      </div>

      <!-- BACK -->
      <div class="card-face back">APQ</div>
    </div>
  </div>
</template>

<script setup>
/* props & event */
  const props = defineProps({
    card: { type: Object, required: true },
  })
  const emit = defineEmits(['flipped'])

  /* ensure each card tracks face‑up state */
  if (props.card.faceUp === undefined) props.card.faceUp = false

  function flip () {
    props.card.faceUp = !props.card.faceUp
    if (props.card.faceUp) emit('flipped', props.card) // notify when turned face‑up
  }
</script>

<style scoped>
.tarot-card {
  width: 90px;
  height: 140px;
  perspective: 1000px;
  cursor: pointer;
  user-select: none;
}

.card-inner {
  width: 100%;
  height: 100%;
  position: relative;
  transition: transform 0.6s;
  transform-style: preserve-3d;
}

.card-inner.flipped {
  transform: rotateY(180deg);
}

.card-face {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  border-radius: 12px;
  padding: 6px;
  border: 2px solid #FFCC00;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.4);
}

/* card front appearance */
.front {
  background: linear-gradient(145deg, #B13BFF 0%, #471396 100%);
  color: #FFCC00;
}

/* card back appearance */
.back {
  background: repeating-linear-gradient(
    45deg,
    #471396 0,
    #471396 10px,
    #3a0f7a 10px,
    #3a0f7a 20px
  );
  color: #FFCC00;
  transform: rotateY(180deg);
}
</style>
