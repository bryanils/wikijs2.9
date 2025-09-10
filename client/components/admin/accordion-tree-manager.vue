<template lang="pug">
  .accordion-tree-manager
    v-row(no-gutters)
      // Tree Management Panel
      v-col(cols='12')
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
                v-list-item(@click='onAddChild(null, "link")')
                  v-list-item-icon
                    v-icon(small) mdi-link
                  v-list-item-title Link
                v-list-item(@click='onAddChild(null, "header")')
                  v-list-item-icon
                    v-icon(small) mdi-format-title
                  v-list-item-title Header
                v-list-item(@click='onAddChild(null, "divider")')
                  v-list-item-icon
                    v-icon(small) mdi-minus
                  v-list-item-title Divider
                v-list-item(@click='onAddChild(null, "accordion")')
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
            .tree-content
              // Show accordion children directly
              template(v-if='currentAccordion && currentAccordion.children && currentAccordion.children.length > 0')
                accordion-tree-item(
                  v-for='child in currentAccordion.children'
                  :key='child.id'
                  :item='child'
                  :selected-item='selectedChild'
                  :expanded-items='expandedItems'
                  @select='onItemSelect'
                  @expand='onItemExpand'
                  @add-child='onAddChild'
                  @remove-child='onRemoveChild'
                  @move-child='onMoveChild'
                  :level='0'
                )
              
              // Show message when accordion has no children
              .text-center.py-4.grey--text(v-else-if='currentAccordion')
                v-icon(color='grey lighten-1') mdi-folder-plus
                .mt-2 This accordion is empty. Add items using the button above.
              
              .text-center.py-8.grey--text(v-else)
                v-icon(large, color='grey lighten-1') mdi-folder-outline
                .mt-2 Select an accordion to manage its structure
                
      //- // Live Preview Panel  
      //- v-col(cols='5')
      //-   v-card(flat, outlined)
      //-     v-toolbar(flat, color='primary', dark, height='48')
      //-       v-icon.mr-2 mdi-eye
      //-       .subtitle-2 Live Preview
      //-       v-spacer
      //-       v-tooltip(top)
      //-         template(v-slot:activator='{ on }')
      //-           v-btn(icon, small, v-on='on', @click='refreshPreview')
      //-             v-icon mdi-refresh
      //-         span Refresh Preview
          
      //-     .preview-container(style='max-height: 500px; overflow-y: auto;')
      //-       v-list.pa-0(v-if='currentAccordion', dense, nav)
      //-         nav-accordion-item(
      //-           :item='previewAccordion'
      //-           :level='0'
      //-         )
            
      //-       .text-center.py-8.grey--text(v-else)
      //-         v-icon(large, color='grey lighten-1') mdi-eye-off
      //-         .mt-2 Preview will appear here
              
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
              v-model='localLabel'
            )
          v-col(cols='3')
            v-text-field(
              outlined
              dense
              label='Icon'
              v-model='localIcon'
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
                v-model='localTarget'
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
                
      // Save/Cancel buttons
      v-card-actions
        v-spacer
        v-btn(text, @click='cancelEdit') Cancel
        v-btn(color='primary', @click='saveChanges') Save Changes
</template>

<script>
import NavAccordionItem from '@/themes/default/components/nav-accordion-item'
import AccordionTreeItem from './accordion-tree-item'

export default {
  name: 'AccordionTreeManager',
  components: {
    NavAccordionItem,
    AccordionTreeItem
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
      expandedItems: {},
      breadcrumbs: [],
      // Local state for text inputs to prevent re-renders
      localLabel: '',
      localIcon: '',
      localTarget: '',
      isEditingChild: false,
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
          // Work directly with the original object - single source of truth
          this.currentAccordion = newVal
          
          // Ensure children array exists
          if (!this.currentAccordion.children) {
            this.$set(this.currentAccordion, 'children', [])
          }
          // Auto-expand the accordion to show its structure
          this.expandedItems[this.currentAccordion.id] = true
          
          // Auto-expand all accordion children so we can see the full structure
          const expandAllAccordions = (items) => {
            if (items) {
              items.forEach(item => {
                if (item.kind === 'accordion' && item.id) {
                  this.expandedItems[item.id] = true
                  if (item.children) {
                    expandAllAccordions(item.children)
                  }
                }
              })
            }
          }
          expandAllAccordions(newVal.children)
        } else {
          this.currentAccordion = null
        }
        this.selectedChild = null
        this.breadcrumbs = []
        // Force update to reflect changes
        this.$nextTick(() => {
          this.$forceUpdate()
        })
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
          this.expandedItems[item.id] = true
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
      this.expandedItems = {}
      this.$forceUpdate()
    },
    
    refreshPreview() {
      this.$forceUpdate()
    },
    
    // Item selection and navigation
    onItemSelect(item) {
      this.selectedChild = item
      // Load values into local state when selecting an item
      this.localLabel = item.label || ''
      this.localIcon = item.icon || ''
      this.localTarget = item.target || ''
      this.isEditingChild = true
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
      if (this.expandedItems[item.id]) {
        delete this.expandedItems[item.id]
      } else {
        this.expandedItems[item.id] = true
      }
    },
    
    // Child management
    onAddChild(parent, kind) {
      // If parent is null, add to the root accordion
      if (!parent) {
        parent = this.currentAccordion
      }
      
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
      this.expandedItems[parent.id] = true
      
      // For deeply nested arrays, we need to trigger reactivity on the root
      // Use $set to ensure Vue detects the change in nested structures
      this.$set(parent, 'children', [...parent.children])
      
      // Don't auto-select the new child - it's annoying
      // this.onItemSelect(newChild)
      
      // Force Vue reactivity for deeply nested changes
      this.$forceUpdate()
      this.emitUpdate()
    },
    
    onRemoveChild(parent, itemToRemove) {
      // If parent is null, we're removing from the root accordion
      if (!parent) {
        parent = this.currentAccordion
      }
      
      if (!parent || !parent.children) {
        return
      }
      
      // Find and remove the item
      const itemIndex = parent.children.findIndex(child => child.id === itemToRemove.id)
      if (itemIndex !== -1) {
        parent.children.splice(itemIndex, 1)
        
        // Force reactivity for deeply nested arrays
        this.$set(parent, 'children', [...parent.children])
        
        if (this.selectedChild && this.selectedChild.id === itemToRemove.id) {
          this.selectedChild = null
          this.breadcrumbs = []
        }
        
        this.$forceUpdate()
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
      // Save the current selection to restore it after update
      const currentSelection = this.selectedChild
      this.emitUpdate()
      // Restore the selection after the update
      this.$nextTick(() => {
        this.selectedChild = currentSelection
      })
    },
    
    // Save/Cancel functionality
    saveChanges() {
      if (this.selectedChild) {
        // Update the original object directly
        this.selectedChild.label = this.localLabel
        this.selectedChild.icon = this.localIcon
        this.selectedChild.target = this.localTarget
        this.isEditingChild = false
        // Notify parent component of the change
        this.emitUpdate()
      }
    },
    
    cancelEdit() {
      // Reset local state to original values
      if (this.selectedChild) {
        this.localLabel = this.selectedChild.label || ''
        this.localIcon = this.selectedChild.icon || ''
        this.localTarget = this.selectedChild.target || ''
      }
      this.isEditingChild = false
    },
    
    selectPageForChild() {
      // This would open the page selector for the selected child
      this.$emit('select-page-for-child', this.selectedChild)
    },
    
    // Called when a page is selected from the page selector
    onPageSelected(path, locale) {
      const target = `/${locale}/${path}`
      // Update both local state and the original object
      this.localTarget = target
      if (this.selectedChild) {
        this.selectedChild.targetType = 'page'
        this.selectedChild.target = target
        // Notify parent of the change
        this.emitUpdate()
      }
    },
    
    // Utilities
    capitalizeFirst(str) {
      return str.charAt(0).toUpperCase() + str.slice(1)
    },
    
    emitUpdate() {
      // Since we're working directly with the original object, we just need to notify parent
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