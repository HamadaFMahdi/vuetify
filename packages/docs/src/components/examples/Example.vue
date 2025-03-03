<template>
  <v-defaults-provider scoped>
    <v-sheet
      border
      class="mb-9 overflow-hidden"
      rounded
    >
      <v-lazy
        v-if="!preview"
        v-model="hasRendered"
        min-height="44"
      >
        <v-toolbar
          :color="isDark ? '#1F1F1F' : 'grey-lighten-4'"
          border="b"
          class="px-1"
          flat
          height="44"
        >
          <v-fade-transition>
            <div v-if="showCode">
              <v-btn
                v-for="(section, i) of sections"
                :key="section.name"
                :active="template === i"
                class="ma-1 text-none"
                variant="text"
                size="small"
                @click="template = i"
              >
                <span :class="template === i ? 'text-high-emphasis' : 'text-medium-emphasis'">
                  {{ upperFirst(section.name) }}
                </span>
              </v-btn>
            </div>
          </v-fade-transition>

          <v-spacer />

          <v-tooltip
            v-for="({ path, ...action }, i) of actions"
            :key="i"
            location="top"
          >
            <template #activator="{ props: tooltip }">
              <v-btn
                v-show="!action.hide"
                class="me-2 text-medium-emphasis"
                density="comfortable"
                variant="text"
                v-bind="mergeProps(action as any, tooltip)"
              />
            </template>

            <span>{{ t(path) }}</span>
          </v-tooltip>
        </v-toolbar>
      </v-lazy>

      <div class="d-flex flex-column">
        <v-expand-transition v-if="hasRendered">
          <v-window v-show="showCode" v-model="template">
            <v-window-item
              v-for="(section, i) of sections"
              :key="section.name"
              :eager="i === 0 || isEager"
            >
              <v-theme-provider :theme="theme">
                <app-markup
                  :code="section.content"
                  :rounded="false"
                />
              </v-theme-provider>
            </v-window-item>
          </v-window>
        </v-expand-transition>

        <v-theme-provider
          :class="showCode && 'border-t'"
          :theme="theme"
          class="pa-2 rounded-b"
          with-background
        >
          <component :is="ExampleComponent" v-if="isLoaded" />
        </v-theme-provider>
      </div>
    </v-sheet>
  </v-defaults-provider>
</template>

<script setup lang="ts">
  // Components
  import ExampleMissing from './ExampleMissing.vue'

  // Composables
  import { useDisplay, useTheme } from 'vuetify'
  import { useI18n } from 'vue-i18n'
  import { usePlayground } from '@/composables/playground'
  import { useUserStore } from '@/store/user'

  // Utilities
  import { computed, mergeProps, onMounted, ref, shallowRef, watch } from 'vue'
  import { getBranch } from '@/util/helpers'
  import { getExample } from 'virtual:examples'
  import { upperFirst } from 'lodash-es'

  const { xs } = useDisplay()
  const { t } = useI18n()
  const userStore = useUserStore()

  const props = defineProps({
    inline: Boolean,
    hideInvert: Boolean,
    file: {
      type: String,
      required: true,
    },
    open: Boolean,
    preview: Boolean,
  })

  function parseTemplate (target: string, template: string) {
    const pattern = {
      composition: /(<script setup>[\w\W]*?<\/script>)/g,
      options: /(<script>[\w\W]*?<\/script>)/g,
    }[target] || new RegExp(`(<${target}(.*)?>[\\w\\W]*<\\/${target}>)`, 'g')
    const parsed = pattern.exec(template)

    return parsed?.[1]
  }

  const isLoaded = ref(false)
  const isError = ref(false)
  const showCode = ref(props.inline || props.open)
  const template = ref(0)
  const hasRendered = ref(false)
  const isEager = shallowRef(false)

  const component = shallowRef()
  const code = ref<string>()
  const ExampleComponent = computed(() => {
    return isError.value ? ExampleMissing : isLoaded.value ? component.value : null
  })
  const sections = computed(() => {
    const _code = code.value
    if (!_code) return []
    const scriptContent = parseTemplate(userStore.composition, _code) ??
      parseTemplate({ composition: 'options', options: 'composition' }[userStore.composition], _code)

    return [
      {
        name: 'template',
        language: 'html',
        content: parseTemplate('template', _code),
      },
      {
        name: 'script',
        language: 'javascript',
        content: scriptContent,
      },
      {
        name: 'style',
        language: 'css',
        content: parseTemplate('style', _code),
      },
    ].filter(v => v.content) as { name: string, content: string, language: string }[]
  })

  onMounted(importExample)

  async function importExample () {
    try {
      const {
        component: _component,
        source: _code,
      } = await getExample(props.file)
      component.value = _component
      code.value = _code
      isLoaded.value = true
      isError.value = false
    } catch (e) {
      console.error(e)
      isLoaded.value = true
      isError.value = true
    }
  }
  const parentTheme = useTheme()
  const _theme = ref<null | string>(null)
  const theme = computed({
    get: () => _theme.value ?? parentTheme.name.value,
    set: val => _theme.value = val,
  })
  const toggleTheme = () => theme.value = theme.value === 'light' ? 'dark' : 'light'

  const isDark = computed(() => {
    return parentTheme.current.value.dark
  })

  const playgroundLink = computed(() => {
    if (!isLoaded.value || isError.value) return null

    const resources = JSON.parse(component.value.playgroundResources || '{}')
    return usePlayground(
      sections.value,
      resources.css,
      resources.imports,
    )
  })

  const actions = computed(() => [
    {
      icon: 'mdi-theme-light-dark',
      path: 'invert-example-colors',
      onClick: toggleTheme,
    },
    {
      icon: '$vuetifyPlay',
      path: 'edit-in-playground',
      href: playgroundLink.value,
      target: '_blank',
      hide: xs.value,
    },
    {
      icon: 'mdi-github',
      path: 'view-in-github',
      href: `https://github.com/vuetifyjs/vuetify/tree/${getBranch()}/packages/docs/src/examples/${props.file}.vue`,
      target: '_blank',
      hide: xs.value,
    },
    {
      icon: 'mdi-clipboard-multiple-outline',
      path: 'copy-example-source',
      onClick: () => {
        navigator.clipboard.writeText(
          sections.value.map(section => section.content).join('\n')
        )
      },
      hide: xs.value,
    },
    {
      icon: !showCode.value ? 'mdi-code-tags' : 'mdi-chevron-up',
      path: !showCode.value ? 'view-source' : 'hide-source',
      onClick: () => {
        showCode.value = !showCode.value
      },
    },
  ])

  watch(showCode, val => val && (isEager.value = true))
</script>
