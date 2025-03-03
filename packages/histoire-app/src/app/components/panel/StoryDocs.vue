<script lang="ts">
import { nextTick, PropType, Ref, ref, toRefs, watch, watchEffect } from 'vue'
import type { Story } from '../../types'
// @ts-expect-error virtual module
import { markdownFiles } from 'virtual:$histoire-markdown-files'
import { histoireConfig } from '../../util/config.js'

export function useStoryDoc (story: Ref<Story>) {
  const renderedDoc = ref('')

  watchEffect(async () => {
    // Markdown file
    const mdKey = story.value.file.filePath.replace(/\.(\w*?)$/, '.md')
    if (markdownFiles[mdKey]) {
      const md = await markdownFiles[mdKey]()
      renderedDoc.value = md.html
      return
    }

    // Custom blocks (Vue)
    // @TODO extract
    let comp = story.value.file?.component
    if (comp) {
      if (comp.__asyncResolved) {
        comp = comp.__asyncResolved
      } else if (comp.__asyncLoader) {
        comp = await comp.__asyncLoader()
      } else if (typeof comp === 'function') {
        try {
          comp = await comp()
        } catch (e) {
          // Noop
          // Could be a class that requires `new com()`
        }
      }
      if (comp?.default) {
        comp = comp.default
      }
      renderedDoc.value = comp.doc
    }
  })

  return {
    renderedDoc,
  }
}
</script>

<script lang="ts" setup>
import { Icon } from '@iconify/vue'
import { useRoute, useRouter } from 'vue-router'
import BaseEmpty from '../base/BaseEmpty.vue'

const props = defineProps({
  story: {
    type: Object as PropType<Story>,
    required: true,
  },
})

const emit = defineEmits<{
  (e: 'scroll-top'): void
}>()

const { story } = toRefs(props)

const { renderedDoc } = useStoryDoc(story)

// Markdown links to other stories
const router = useRouter()

// we are just using URL to parse the pathname and hash - the base doesn't
// matter and is only passed to support same-host hrefs.
const fakeHost = `http://a.com`

function onClick (e: MouseEvent) {
  const link = (e.target as Element).closest('a')
  if (
    link &&
    link.getAttribute('data-route') &&
    !e.ctrlKey &&
    !e.shiftKey &&
    !e.altKey &&
    !e.metaKey &&
    link.target !== `_blank`
  ) {
    e.preventDefault()
    const url = new URL(link.href, fakeHost)
    const targetHref = url.pathname + url.search + url.hash
    router.push(targetHref)
  }
}

// Handle URL anchor

function getHash () {
  const hash = location.hash
  if (histoireConfig.routerMode === 'hash') {
    const index = hash.indexOf('#', 1)
    if (index !== -1) {
      return hash.slice(index)
    } else {
      return undefined
    }
  }
  return hash
}

async function scrollToAnchor () {
  await nextTick()
  const hash = getHash()
  if (hash) {
    const anchor = document.querySelector(hash)
    if (anchor) {
      anchor.scrollIntoView()
      return
    }
  }
  emit('scroll-top')
}

watch(renderedDoc, () => {
  scrollToAnchor()
}, {
  immediate: true,
})

// Fix anchor links with router mode 'hash'

const renderedEl = ref<HTMLElement>()
const route = useRoute()

async function patchAnchorLinks () {
  await nextTick()
  if (histoireConfig.routerMode === 'hash' && renderedEl.value) {
    const links = renderedEl.value.querySelectorAll('a.header-anchor')
    for (const link of links) {
      const href = link.getAttribute('href')
      if (href) {
        link.setAttribute('href', `#${route.path + href}`)
      }
    }
  }
}

watch(renderedDoc, () => {
  patchAnchorLinks()
}, {
  immediate: true,
})
</script>

<template>
  <div
    class="histoire-story-docs"
    @click.capture="onClick"
  >
    <BaseEmpty
      v-if="!renderedDoc"
    >
      <Icon
        icon="carbon:document-unknown"
        class="htw-w-8 htw-h-8 htw-opacity-50 htw-mb-6"
      />
      No documentation available
    </BaseEmpty>
    <div
      v-else
      ref="renderedEl"
      class="htw-prose dark:htw-prose-invert htw-p-4 htw-max-w-none"
      data-test-id="story-docs"
      v-html="renderedDoc"
    />
  </div>
</template>
