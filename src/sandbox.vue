<script setup>
  import { ref, reactive, watch } from 'vue'
  import ContentReference from './content-reference.vue';

  const TRANSLATION_DOMAIN = 'f74e9cb3-2b53-4c85-9b0c-f1d61b032b3f.localhost:5889'

  const myStuff = reactive(await Agent.state('my-content'))
  const activeItem = ref(null)
  const activeItemData = ref(null)
  const translatedActiveItemData = ref(null)
  const translationLanguage = ref('en')

  async function translatedObject(id, lang) {
    const translations = await Agent.query('translate-item', [id, [lang]], TRANSLATION_DOMAIN)
    let translated = JSON.parse(JSON.stringify(await Agent.state(id)))

    translations
      .forEach(({ path, value }) => {
        let ref = translated
        const p = path.slice(1)
        while (p.length > 1 && ref[p[0]]) ref = ref[p.shift()]
        ref[p[0]] = value
      })

      delete translated.translations

      return translated
  }

  watch(
    () => activeItem.value,
    async (val) => {
      if (!val) return
      activeItemData.value = await Agent.state(val)
    }
  )

  watch(
    () => [activeItemData.value, translationLanguage.value],
    async () => {
      const id = activeItem.value
      const lang = translationLanguage.value
      translatedActiveItemData.value = await translatedObject(id, [lang])
    }
  )

  async function addNew() {
    const newId = await Agent.create({
      active: {
        name: "Multiple Choice Question",
        question: 'What is the third planet from the sun?',
        options: [
          'Mars',
          'Venus',
          'Earth',
          'Which sun?'
        ],
        hint: 'The sun that orbits earth.'
      }
    })
    myStuff[newId] = true
    activeItem.value = newId
  }

  async function addTranslations() {
    if (!activeItem.value) return

  const state = await Agent.state(activeItem.value)
  state.translations = { source_language: 'en-us', paths: [] }
  Object.entries(state)
    .forEach(([key, val]) => {
      if (typeof val === 'string') {
        state.translations.paths.push([key]) 
      } else if (Array.isArray(val)) {
        val.forEach((v,i) => {
          if (typeof v === 'string') state.translations.paths.push([key, i])        
        })
      }
    }) 
  }

</script>

<template>
  <h1>Sandbox</h1>
  <h3>My Stuff</h3>
  <div>
    <div
        v-for="id in Object.keys(myStuff)"
        :class="id === activeItem ? 'active': ''"
        @click="activeItem = id"
        @click.shift="delete myStuff[id]"
    >
      <ContentReference :id="id" :key="id"/> {{ id }}
    </div>
  </div>
  <button @click="addTranslations" :disabled="!activeItem">Add Translations</button>
  <button @click="addNew">Add New</button>
  <hr>
  <div class="translation-container">
    <pre>{{ activeItemData }}</pre>
    <pre><input v-model="translationLanguage" />
{{  translatedActiveItemData  }}</pre>
  </div>
</template>

<style scoped>
.active { background: chartreuse; }

.translation-container {
  display: flex;
}

.translation-container>* {
  flex-grow: 1;
}
</style>