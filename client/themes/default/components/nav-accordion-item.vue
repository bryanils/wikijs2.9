<template lang="pug">
  div
    v-list-item(
      v-if='item.k === `link`'
      :href='item.t'
      :target='item.y === `externalblank` ? `_blank` : `_self`'
      :rel='item.y === `externalblank` ? `noopener` : ``'
      :style='`padding-left: ` + (level * 16 + 16) + `px;`'
      )
      v-list-item-avatar(size='20', tile)
        v-icon(v-if='item.c && item.c.match(/fa[a-z] fa-/)', size='16') {{ item.c }}
        v-icon(v-else-if='item.c', size='16') {{ item.c }}
        v-icon(v-else, size='16') mdi-file-document-outline
      v-list-item-title {{ item.l }}
    v-divider.my-1(
      v-else-if='item.k === `divider`'
      :style='`margin-left: ` + (level * 16 + 16) + `px;`'
      )
    v-subheader(
      v-else-if='item.k === `header`'
      :style='`padding-left: ` + (level * 16 + 16) + `px;`'
      ) {{ item.l }}
    template(v-else-if='item.k === `accordion`')
      v-list-item(
        @click='toggleExpanded'
        :style='`padding-left: ` + (level * 16 + 16) + `px;`'
      )
        v-list-item-avatar(size='20', tile)
          v-icon(v-if='item.c && item.c.match(/fa[a-z] fa-/)', size='16') {{ item.c }}
          v-icon(v-else) {{ item.c || 'mdi-folder' }}
        v-list-item-title {{ item.l }}
        v-list-item-action
          v-icon {{ isExpanded ? 'mdi-chevron-up' : 'mdi-chevron-down' }}
      nav-accordion-item(
        v-if='isExpanded'
        v-for='(subItem, subIndex) of item.children || []'
        :key='`nested-` + level + `-` + subIndex'
        :item='subItem'
        :level='level + 1'
        )
</template>

<script>
export default {
  name: 'NavAccordionItem',
  props: {
    item: {
      type: Object,
      required: true
    },
    level: {
      type: Number,
      default: 0
    }
  },
  data() {
    return {
      internalExpanded: true
    }
  },
  computed: {
    isExpanded() {
      return this.item.k === 'accordion' ? this.internalExpanded : false
    }
  },
  methods: {
    toggleExpanded() {
      this.internalExpanded = !this.internalExpanded
      const storageKey = `accordion-${this.item.i || this.item.l}`
      localStorage.setItem(storageKey, this.internalExpanded.toString())
    }
  },
  mounted() {
    if (this.item.k === 'accordion') {
      const storageKey = `accordion-${this.item.i || this.item.l}`
      const stored = localStorage.getItem(storageKey)
      if (stored !== null) {
        this.internalExpanded = stored === 'true'
      } else {
        this.internalExpanded = this.item.expanded !== false
      }
    }
  }
}
</script>