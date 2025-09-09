<template lang="pug">
  .accordion-tree-item
    // Main item
    .tree-item(
      :class='{ "selected": selectedItem && selectedItem.id === item.id, "is-accordion": item.kind === "accordion" }'
      @click='selectItem'
    )
      .d-flex.align-center.py-2.px-3
        // Expand/collapse button for accordions with children
        v-btn(
          v-if='item.kind === "accordion" && hasChildren'
          icon
          x-small
          @click.stop='toggleExpanded'
        )
          v-icon(small) {{ isExpanded ? 'mdi-chevron-down' : 'mdi-chevron-right' }}
        
        .spacer(v-else, style='width: 24px;')
        
        // Icon
        v-icon.mr-3(:color='getItemColor(item.kind)', small) {{ getItemIcon(item) }}
        
        // Label
        .flex-grow-1.item-label {{ item.label }}
        
        // Kind indicator
        v-chip.mr-2(
          x-small
          :color='getItemColor(item.kind)'
          text-color='white'
          style='font-size: 10px;'
        ) {{ item.kind.toUpperCase() }}
        
        // Actions menu
        v-menu(offset-y, left)
          template(v-slot:activator='{ on }')
            v-btn(icon, x-small, v-on='on', @click.stop)
              v-icon(small) mdi-dots-vertical
          v-list(dense)
            v-list-item(@click='$emit("select", item)')
              v-list-item-icon
                v-icon(small) mdi-pencil
              v-list-item-title Edit
            v-divider(v-if='item.kind === "accordion"')
            template(v-if='item.kind === "accordion"')
              v-list-item(@click='addChild("link")')
                v-list-item-icon
                  v-icon(small) mdi-link
                v-list-item-title Add Link
              v-list-item(@click='addChild("header")')
                v-list-item-icon
                  v-icon(small) mdi-format-title
                v-list-item-title Add Header
              v-list-item(@click='addChild("divider")')
                v-list-item-icon
                  v-icon(small) mdi-minus
                v-list-item-title Add Divider
              v-list-item(@click='addChild("accordion")')
                v-list-item-icon
                  v-icon(small) mdi-folder
                v-list-item-title Add Nested Accordion
              v-divider
            v-list-item(@click='duplicateItem')
              v-list-item-icon
                v-icon(small) mdi-content-duplicate
              v-list-item-title Duplicate
            v-list-item(@click='deleteItem', class='error--text')
              v-list-item-icon
                v-icon(small, color='error') mdi-delete
              v-list-item-title Delete
    
    // Children (if accordion and expanded)
    .tree-children(
      v-if='item.kind === "accordion" && hasChildren && isExpanded'
      style='margin-left: 20px; border-left: 2px solid #e0e0e0;'
    )
      draggable(
        v-model='item.children'
        group='accordion-items'
        :animation='200'
        ghost-class='ghost-item'
        chosen-class='chosen-item'
        drag-class='drag-item'
        @change='onChildrenChange'
      )
        accordion-tree-item(
          v-for='(child, index) in item.children'
          :key='child.id'
          :item='child'
          :selected-item='selectedItem'
          :expanded-items='expandedItems'
          @select='$emit("select", $event)'
          @expand='$emit("expand", $event)'
          @add-child='$emit("add-child", $event.parent, $event.kind)'
          @remove-child='$emit("remove-child", item, index)'
          @move-child='$emit("move-child", $event)'
        )
    
    // Empty state for accordion with no children
    .tree-empty(
      v-else-if='item.kind === "accordion" && !hasChildren && isExpanded'
      style='margin-left: 20px;'
    )
      .text-center.py-4.grey--text(style='border: 2px dashed #e0e0e0; border-radius: 4px;')
        v-icon(color='grey lighten-1') mdi-folder-open-outline
        .caption.mt-1 No items yet
        v-menu(offset-y)
          template(v-slot:activator='{ on }')
            v-btn.mt-2(small, outlined, color='primary', v-on='on')
              v-icon(left, small) mdi-plus
              span Add Item
          v-list(dense)
            v-list-item(@click='addChild("link")')
              v-list-item-icon
                v-icon(small) mdi-link
              v-list-item-title Link
            v-list-item(@click='addChild("header")')
              v-list-item-icon
                v-icon(small) mdi-format-title
              v-list-item-title Header
            v-list-item(@click='addChild("divider")')
              v-list-item-icon
                v-icon(small) mdi-minus
              v-list-item-title Divider
            v-list-item(@click='addChild("accordion")')
              v-list-item-icon
                v-icon(small) mdi-folder
              v-list-item-title Nested Accordion
</template>

<script>
import draggable from 'vuedraggable'

export default {
  name: 'AccordionTreeItem',
  components: {
    draggable
  },
  props: {
    item: {
      type: Object,
      required: true
    },
    selectedItem: {
      type: Object,
      default: null
    },
    expandedItems: {
      type: Set,
      default: () => new Set()
    }
  },
  computed: {
    hasChildren() {
      return this.item.children && this.item.children.length > 0
    },
    
    isExpanded() {
      return this.expandedItems.has(this.item.id)
    }
  },
  methods: {
    selectItem() {
      this.$emit('select', this.item)
    },
    
    toggleExpanded() {
      this.$emit('expand', this.item)
    },
    
    addChild(kind) {
      this.$emit('add-child', this.item, kind)
    },
    
    duplicateItem() {
      const duplicate = {
        ...this.item,
        id: `temp_${Date.now()}`,
        label: `${this.item.label} (Copy)`
      }
      
      if (this.item.children) {
        duplicate.children = [...this.item.children]
      }
      
      this.$emit('duplicate-item', duplicate)
    },
    
    deleteItem() {
      this.$emit('remove-child', null, this.item)
    },
    
    getItemIcon(item) {
      const icons = {
        accordion: item.icon || 'mdi-folder',
        link: item.icon || 'mdi-link',
        header: item.icon || 'mdi-format-title',
        divider: 'mdi-minus'
      }
      return icons[item.kind] || 'mdi-circle'
    },
    
    getItemColor(kind) {
      const colors = {
        accordion: 'orange',
        link: 'blue',
        header: 'purple',
        divider: 'grey'
      }
      return colors[kind] || 'grey'
    },
    
    onChildrenChange(event) {
      if (event.moved) {
        this.$emit('move-child', {
          parent: this.item,
          oldIndex: event.moved.oldIndex,
          newIndex: event.moved.newIndex
        })
      }
      
      if (event.added) {
        this.$emit('move-child', {
          toParent: this.item,
          toIndex: event.added.newIndex,
          item: event.added.element
        })
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.accordion-tree-item {
  .tree-item {
    cursor: pointer;
    border-radius: 4px;
    transition: background-color 0.2s;
    
    &:hover {
      background-color: rgba(0, 0, 0, 0.04);
    }
    
    &.selected {
      background-color: rgba(33, 150, 243, 0.1);
      border-left: 3px solid #2196f3;
    }
    
    &.is-accordion {
      background-color: rgba(255, 152, 0, 0.05);
      
      &:hover {
        background-color: rgba(255, 152, 0, 0.1);
      }
      
      &.selected {
        background-color: rgba(255, 152, 0, 0.15);
        border-left-color: #ff9800;
      }
    }
  }
  
  .item-label {
    font-weight: 500;
    font-size: 14px;
  }
  
  .tree-children {
    margin-top: 4px;
    margin-bottom: 4px;
  }
  
  .tree-empty {
    margin-top: 8px;
    margin-bottom: 8px;
  }
}

// Drag and drop styles
.ghost-item {
  opacity: 0.5;
  background-color: #e3f2fd !important;
}

.chosen-item {
  background-color: #bbdefb !important;
}

.drag-item {
  transform: rotate(5deg);
  background-color: #2196f3 !important;
  color: white !important;
}
</style>