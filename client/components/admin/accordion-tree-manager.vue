<template lang="pug">
  .accordion-tree-manager
    v-row(no-gutters)
      // Tree Management Panel
      v-col(cols='7')
        v-card(flat, outlined)
          v-toolbar(flat, color='grey lighten-4', height='48')
            v-icon.mr-2 mdi-file-tree
            .subtitle-2 Accordion Structure
            v-spacer
            v-menu(offset-y)
              template(v-slot:activator='{ on }')
                v-btn(small, color='primary', v-on='on')
                  v-icon(left, small) mdi-plus
                  span Add Item
              v-list(dense)
                v-list-item(@click='onAddChild(currentAccordion, "link")')
                  v-list-item-icon
                    v-icon(small) mdi-link
                  v-list-item-title Link
                v-list-item(@click='onAddChild(currentAccordion, "header")')
                  v-list-item-icon
                    v-icon(small) mdi-format-title
                  v-list-item-title Header
                v-list-item(@click='onAddChild(currentAccordion, "divider")')
                  v-list-item-icon
                    v-icon(small) mdi-minus
                  v-list-item-title Divider
                v-list-item(@click='onAddChild(currentAccordion, "accordion")')
                  v-list-item-icon
                    v-icon(small) mdi-folder
                  v-list-item-title Nested Accordion
            v-tooltip(top)
              template(v-slot:activator='{ on }')
                v-btn(icon, small, v-on='on', @click='expandAll')
                  v-icon mdi-unfold-more-horizontal
              span Expand All
            v-tooltip(top)
              template(v-slot:activator='{ on }')
                v-btn(icon, small, v-on='on', @click='collapseAll')
                  v-icon mdi-unfold-less-horizontal
              span Collapse All
          
          .tree-container(style='max-height: 500px; overflow-y: auto;')
            // Breadcrumb if editing nested item
            v-breadcrumbs.py-2.px-3(v-if='breadcrumbs.length > 0', small)
              v-breadcrumbs-item(v-for='(crumb, idx) in breadcrumbs', :key='idx', @click='navigateTo(crumb.item)')
                v-icon(small, left) {{ crumb.icon }}
                | {{ crumb.label }}
                
            // Accordion Tree Structure
            .tree-content.pa-3
              accordion-tree-item(
                v-if='currentAccordion'
                :item='currentAccordion'
                :selected-item='selectedChild'
                :expanded-items='expandedItems'
                @select='onItemSelect'
                @expand='onItemExpand'
                @add-child='onAddChild'
                @remove-child='onRemoveChild'
                @move-child='onMoveChild'
              )
              
              .text-center.py-8.grey--text(v-else)
                v-icon(large, color='grey lighten-1') mdi-folder-outline
                .mt-2 Select an accordion to manage its structure
                
      // Live Preview Panel  
      v-col(cols='5')
        v-card(flat, outlined)
          v-toolbar(flat, color='primary', dark, height='48')
            v-icon.mr-2 mdi-eye
            .subtitle-2 Live Preview
            v-spacer
            v-tooltip(top)
              template(v-slot:activator='{ on }')
                v-btn(icon, small, v-on='on', @click='refreshPreview')
                  v-icon mdi-refresh
              span Refresh Preview
          
          .preview-container(style='max-height: 500px; overflow-y: auto;')
            v-list.pa-0(v-if='currentAccordion', dense, nav)
              nav-accordion-item(
                :item='previewAccordion'
                :level='0'
              )
            
            .text-center.py-8.grey--text(v-else)
              v-icon(large, color='grey lighten-1') mdi-eye-off
              .mt-2 Preview will appear here
              
    // Child Item Editor
    v-card.mt-4(v-if='selectedChild', outlined)
      v-toolbar(flat, color='teal lighten-1', dark, height='48')
        v-icon.mr-2 mdi-pencil
        .subtitle-2 Edit {{ selectedChild.kind === 'accordion' ? 'Nested Accordion' : capitalizeFirst(selectedChild.kind) }}
        v-spacer
        v-btn(icon, small, @click='selectedChild = null')
          v-icon mdi-close
          
      v-card-text.pb-2
        v-row
          v-col(cols='6')
            v-text-field(
              outlined
              dense
              label='Label'
              v-model='selectedChild.label'
              @input='onChildUpdate'
            )
          v-col(cols='3')
            v-text-field(
              outlined
              dense
              label='Icon'
              v-model='selectedChild.icon'
              @input='onChildUpdate'
            )
          v-col(cols='3', v-if='selectedChild.kind === "accordion"')
            v-switch(
              v-model='selectedChild.expanded'
              label='Expanded'
              @change='onChildUpdate'
              hide-details
            )
              
        // Link-specific fields
        template(v-if='selectedChild.kind === "link"')
          v-row
            v-col(cols='4')
              v-select(
                outlined
                dense
                label='Target Type'
                v-model='selectedChild.targetType'
                :items='targetTypes'
                @change='onChildUpdate'
              )
            v-col(cols='8')
              v-text-field(
                v-if='selectedChild.targetType === "external" || selectedChild.targetType === "externalblank"'
                outlined
                dense
                label='URL'
                v-model='selectedChild.target'
                @input='onChildUpdate'
              )
              .d-flex.align-center(v-else-if='selectedChild.targetType === "page"')
                v-btn(
                  color='primary'
                  small
                  @click='selectPageForChild'
                )
                  v-icon(left, small) mdi-magnify
                  span Select Page
                .ml-3.caption.primary--text {{ selectedChild.target || 'No page selected' }}
</template>

<script>
import NavAccordionItem from '@/themes/default/components/nav-accordion-item'

export default {
  name: 'AccordionTreeManager',
  components: {
    NavAccordionItem,
    AccordionTreeItem: () => import('./accordion-tree-item')
  },
  props: {
    accordion: {
      type: Object,
      default: () => ({})
    },
    value: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {
      currentAccordion: null,
      selectedChild: null,
      expandedItems: new Set(),
      breadcrumbs: [],
      targetTypes: [
        { text: 'External Link', value: 'external' },
        { text: 'External Link (New Tab)', value: 'externalblank' },
        { text: 'Home Page', value: 'home' },
        { text: 'Wiki Page', value: 'page' }
      ]
    }
  },
  computed: {
    previewAccordion() {
      if (!this.currentAccordion) return null
      
      // Transform admin accordion to preview format
      return this.transformToPreviewFormat(this.currentAccordion)
    }
  },
  watch: {
    accordion: {
      immediate: true,
      deep: true,
      handler(newVal) {
        if (newVal && newVal.id) {
          this.currentAccordion = { ...newVal }
        } else {
          this.currentAccordion = null
        }
        this.selectedChild = null
        this.breadcrumbs = []
      }
    }
  },
  methods: {
    // Transform admin format to preview format
    transformToPreviewFormat(item) {
      const transformed = {
        i: item.id,
        k: item.kind,
        l: item.label,
        c: item.icon,
        expanded: item.expanded
      }
      
      if (item.kind === 'link') {
        transformed.y = item.targetType
        transformed.t = item.target
      }
      
      if (item.children && item.children.length > 0) {
        transformed.children = item.children.map(child => 
          this.transformToPreviewFormat(child)
        )
      }
      
      return transformed
    },
    
    // Tree management
    expandAll() {
      const expandItem = (item) => {
        if (item.id) {
          this.expandedItems.add(item.id)
        }
        if (item.children) {
          item.children.forEach(expandItem)
        }
      }
      
      if (this.currentAccordion) {
        expandItem(this.currentAccordion)
      }
      this.$forceUpdate()
    },
    
    collapseAll() {
      this.expandedItems.clear()
      this.$forceUpdate()
    },
    
    refreshPreview() {
      this.$forceUpdate()
    },
    
    // Item selection and navigation
    onItemSelect(item) {
      this.selectedChild = item
      this.buildBreadcrumbs(item)
    },
    
    navigateTo(item) {
      this.onItemSelect(item)
    },
    
    buildBreadcrumbs(targetItem) {
      const breadcrumbs = []
      
      const findPath = (current, target, path = []) => {
        if (current.id === target.id) {
          this.breadcrumbs = [...path, {
            label: current.label,
            icon: this.getIconForKind(current.kind),
            item: current
          }]
          return true
        }
        
        if (current.children) {
          for (const child of current.children) {
            const newPath = [...path, {
              label: current.label,
              icon: this.getIconForKind(current.kind), 
              item: current
            }]
            if (findPath(child, target, newPath)) {
              return true
            }
          }
        }
        return false
      }
      
      if (this.currentAccordion && targetItem.id !== this.currentAccordion.id) {
        findPath(this.currentAccordion, targetItem)
      } else {
        this.breadcrumbs = []
      }
    },
    
    getIconForKind(kind) {
      const icons = {
        accordion: 'mdi-folder',
        link: 'mdi-link',
        header: 'mdi-format-title',
        divider: 'mdi-minus'
      }
      return icons[kind] || 'mdi-circle'
    },
    
    // Tree expansion
    onItemExpand(item) {
      if (this.expandedItems.has(item.id)) {
        this.expandedItems.delete(item.id)
      } else {
        this.expandedItems.add(item.id)
      }
      this.$forceUpdate()
    },
    
    // Child management
    onAddChild(parent, kind) {
      if (!parent || !parent.children) {
        if (parent) {
          this.$set(parent, 'children', [])
        } else {
          return
        }
      }
      
      const newChild = {
        id: `temp_${Date.now()}_${Math.random()}`,
        kind,
        label: `New ${this.capitalizeFirst(kind)}`,
        visibilityMode: 'all',
        visibilityGroups: []
      }
      
      if (kind === 'link') {
        newChild.icon = 'mdi-link'
        newChild.targetType = 'page'
        newChild.target = ''
      } else if (kind === 'header') {
        newChild.icon = 'mdi-format-title'
      } else if (kind === 'divider') {
        newChild.icon = 'mdi-minus'
        newChild.label = ''
      } else if (kind === 'accordion') {
        newChild.icon = 'mdi-folder'
        newChild.expanded = false
        newChild.children = []
      }
      
      parent.children.push(newChild)
      this.expandedItems.add(parent.id)
      this.onItemSelect(newChild)
      this.emitUpdate()
    },
    
    onRemoveChild(parent, childIndex) {
      if (parent.children && parent.children[childIndex]) {
        const removedChild = parent.children[childIndex]
        parent.children.splice(childIndex, 1)
        
        if (this.selectedChild && this.selectedChild.id === removedChild.id) {
          this.selectedChild = null
          this.breadcrumbs = []
        }
        
        this.emitUpdate()
      }
    },
    
    onMoveChild(fromParent, toParent, fromIndex, toIndex) {
      const child = fromParent.children.splice(fromIndex, 1)[0]
      
      if (!toParent.children) {
        this.$set(toParent, 'children', [])
      }
      
      toParent.children.splice(toIndex, 0, child)
      this.emitUpdate()
    },
    
    // Child editing
    onChildUpdate() {
      this.emitUpdate()
    },
    
    selectPageForChild() {
      // This would open the page selector for the selected child
      this.$emit('select-page-for-child', this.selectedChild)
    },
    
    // Utilities
    capitalizeFirst(str) {
      return str.charAt(0).toUpperCase() + str.slice(1)
    },
    
    emitUpdate() {
      this.$emit('input', this.currentAccordion)
      this.$emit('change', this.currentAccordion)
    }
  }
}
</script>

<style lang="scss" scoped>
.accordion-tree-manager {
  .tree-container {
    border-top: 1px solid #e0e0e0;
    background-color: #fafafa;
  }
  
  .preview-container {
    border-top: 1px solid #e0e0e0;
    background-color: #f5f5f5;
  }
  
  .tree-content {
    min-height: 200px;
  }
}
</style>