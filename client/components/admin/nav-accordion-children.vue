<template lang="pug">
  div
    template(v-for='child in children')
      v-list-item(
        v-if='child.kind === "link"'
        :key='child.id'
        :class='(child === current) ? "blue" : ""'
        @click='$emit("select", child)'
        :style='`padding-left: ${(level * 20) + 40}px;`'
      )
        v-list-item-avatar(size='20', tile)
          v-icon(v-if='child.icon && child.icon.match(/fa[a-z] fa-/)', size='16') {{ child.icon }}
          v-icon(v-else, size='16') {{ child.icon || 'mdi-link' }}
        v-list-item-title.caption {{ child.label }}
      
      .py-1.clickable(
        v-else-if='child.kind === "divider"'
        :key='child.id'
        :class='(child === current) ? "blue" : ""'
        @click='$emit("select", child)'
        :style='`padding-left: ${(level * 20) + 40}px;`'
      )
        v-divider
      
      v-subheader.clickable(
        v-else-if='child.kind === "header"'
        :key='child.id'
        :class='(child === current) ? "blue" : ""'
        @click='$emit("select", child)'
        :style='`padding-left: ${(level * 20) + 40}px; font-size: 12px;`'
      ) {{ child.label }}
      
      template(v-else-if='child.kind === "accordion"')
        v-list-item(
          :key='child.id'
          :class='(child === current) ? "blue" : ""'
          @click='$emit("toggle-accordion", child)'
          :style='`padding-left: ${(level * 20) + 40}px;`'
        )
          v-list-item-avatar(size='20', tile)
            v-icon(v-if='child.icon && child.icon.match(/fa[a-z] fa-/)', size='16') {{ child.icon }}
            v-icon(v-else, size='16') {{ child.icon || 'mdi-folder' }}
          v-list-item-title.caption {{ child.label }}
          v-list-item-action
            v-icon(size='16') {{ expandedAccordions && expandedAccordions.has(child.id) ? 'mdi-chevron-up' : 'mdi-chevron-down' }}
        
        // Recursively show nested accordion children when expanded
        nav-accordion-children(
          v-if='expandedAccordions && expandedAccordions.has(child.id) && child.children && child.children.length > 0'
          :children='child.children'
          :current='current'
          :level='level + 1'
          :expanded-accordions='expandedAccordions'
          @select='$emit("select", $event)'
          @toggle-accordion='$emit("toggle-accordion", $event)'
        )
</template>

<script>
export default {
  name: 'NavAccordionChildren',
  props: {
    children: {
      type: Array,
      required: true
    },
    current: {
      type: Object,
      default: () => ({})
    },
    level: {
      type: Number,
      default: 0
    },
    expandedAccordions: {
      type: Set,
      default: () => new Set()
    }
  }
}
</script>

<style lang='scss' scoped>
.clickable {
  cursor: pointer;

  &:hover {
    background-color: rgba(33, 150, 243, 0.25);
  }
}
</style>