<template lang="pug">
  v-dialog(v-model='dialog', max-width='900px', persistent, scrollable)
    v-card(style='height: 700px;')
      v-toolbar(flat, color='orange', dark)
        v-icon.mr-2 mdi-auto-fix
        .subtitle-1 Accordion Creation Wizard
        v-spacer
        v-btn(icon, @click='close')
          v-icon mdi-close
          
      v-card-text.pa-0
        v-stepper(v-model='currentStep', vertical, style='height: 100%;')
          // Step 1: Basic Information
          v-stepper-step(:complete='currentStep > 1', step='1', color='orange')
            | Basic Information
            small Configure accordion name and appearance
          v-stepper-content(step='1')
            v-card(flat)
              v-card-text
                v-row
                  v-col(cols='8')
                    v-text-field(
                      v-model='accordion.label'
                      label='Accordion Name'
                      placeholder='e.g. User Guide, API Reference'
                      outlined
                      prepend-icon='mdi-format-title'
                      :rules='[v => !!v || "Name is required"]'
                    )
                  v-col(cols='4')
                    v-text-field(
                      v-model='accordion.icon'
                      label='Icon'
                      placeholder='mdi-folder'
                      outlined
                      prepend-icon='mdi-dice-5'
                    )
                v-switch(
                  v-model='accordion.expanded'
                  label='Expanded by default'
                  color='orange'
                )
              v-card-actions
                v-spacer
                v-btn(color='orange', @click='currentStep = 2', :disabled='!accordion.label') Continue
                  
          // Step 2: Structure Template
          v-stepper-step(:complete='currentStep > 2', step='2', color='orange')
            | Choose Template
            small Select a pre-defined structure or start blank
          v-stepper-content(step='2')
            v-card(flat)
              v-card-text
                .subtitle-2.mb-4 Select a template to get started quickly:
                v-row
                  v-col(cols='6', v-for='template in templates', :key='template.id')
                    v-card(
                      outlined
                      :color='selectedTemplate === template.id ? "orange lighten-4" : ""'
                      :class='selectedTemplate === template.id ? "orange--border" : ""'
                      @click='selectTemplate(template.id)'
                      style='cursor: pointer;'
                    )
                      v-card-text.text-center.py-6
                        v-icon.mb-2(large, :color='template.color') {{ template.icon }}
                        .subtitle-2 {{ template.name }}
                        .caption.grey--text {{ template.description }}
                        v-chip.mt-2(small, :color='template.color', text-color='white')
                          | {{ template.items.length }} items
              v-card-actions
                v-btn(text, @click='currentStep = 1') Back
                v-spacer
                v-btn(color='orange', @click='currentStep = 3') Continue
                  
          // Step 3: Add Content
          v-stepper-step(:complete='currentStep > 3', step='3', color='orange')
            | Add Content
            small Add pages and organize structure
          v-stepper-content(step='3')
            v-card(flat, style='height: 400px;')
              v-card-text.pa-0
                v-row(no-gutters, style='height: 100%;')
                  // Content Builder
                  v-col(cols='7', style='border-right: 1px solid #e0e0e0;')
                    .d-flex.align-center.pa-3(style='background: #f5f5f5; border-bottom: 1px solid #e0e0e0;')
                      .subtitle-2 Accordion Structure
                      v-spacer
                      v-btn(small, outlined, color='primary', @click='openBulkPageSelector')
                        v-icon(left, small) mdi-file-multiple
                        | Add Multiple Pages
                        
                    .content-builder.pa-3(style='height: calc(100% - 60px); overflow-y: auto;')
                      draggable(v-model='accordion.children', group='wizard-items', :animation='200')
                        .wizard-item(
                          v-for='(item, index) in accordion.children'
                          :key='item.id'
                          :class='{ "selected": selectedItem === item }'
                          @click='selectItem(item)'
                        )
                          .d-flex.align-center.pa-2
                            v-icon.mr-3(small, :color='getItemColor(item.kind)') {{ getItemIcon(item) }}
                            .flex-grow-1 {{ item.label }}
                            v-chip.mr-2(x-small, :color='getItemColor(item.kind)', text-color='white') {{ item.kind.toUpperCase() }}
                            v-btn(icon, x-small, @click.stop='removeItem(index)')
                              v-icon mdi-delete
                              
                      .text-center.py-4(v-if='accordion.children.length === 0')
                        .grey--text No items yet
                        
                      // Add item menu
                      .mt-4
                        v-menu(offset-y)
                          template(v-slot:activator='{ on }')
                            v-btn(v-on='on', small, color='primary', outlined)
                              v-icon(left, small) mdi-plus
                              | Add Item
                          v-list(dense)
                            v-list-item(@click='addWizardItem("link")')
                              v-list-item-icon
                                v-icon(small) mdi-link
                              v-list-item-title Link
                            v-list-item(@click='addWizardItem("header")')
                              v-list-item-icon
                                v-icon(small) mdi-format-title
                              v-list-item-title Header
                            v-list-item(@click='addWizardItem("divider")')
                              v-list-item-icon
                                v-icon(small) mdi-minus
                              v-list-item-title Divider
                  
                  // Item Editor
                  v-col(cols='5')
                    .d-flex.align-center.pa-3(style='background: #f5f5f5; border-bottom: 1px solid #e0e0e0;')
                      .subtitle-2 
                        template(v-if='selectedItem') Edit {{ capitalizeFirst(selectedItem.kind) }}
                        template(v-else) Select an item to edit
                        
                    .item-editor.pa-3(style='height: calc(100% - 60px); overflow-y: auto;')
                      template(v-if='selectedItem')
                        v-text-field(
                          v-model='selectedItem.label'
                          label='Label'
                          outlined
                          dense
                        )
                        v-text-field(
                          v-model='selectedItem.icon'
                          label='Icon'
                          outlined
                          dense
                        )
                        template(v-if='selectedItem.kind === "link"')
                          v-select(
                            v-model='selectedItem.targetType'
                            :items='targetTypes'
                            label='Target Type'
                            outlined
                            dense
                          )
                          v-btn.mt-3(
                            v-if='selectedItem.targetType === "page"'
                            color='primary'
                            small
                            block
                            @click='selectPageForWizardItem'
                          )
                            v-icon(left, small) mdi-magnify
                            | Select Page
                          .caption.mt-2.primary--text(v-if='selectedItem.target') {{ selectedItem.target }}
                      .text-center.py-8.grey--text(v-else)
                        v-icon(large, color='grey lighten-1') mdi-cursor-default-click
                        .mt-2 Click an item to edit it
                        
              v-card-actions
                v-btn(text, @click='currentStep = 2') Back
                v-spacer
                v-btn(color='orange', @click='currentStep = 4') Continue
                  
          // Step 4: Review & Create
          v-stepper-step(:complete='currentStep > 4', step='4', color='orange')
            | Review & Create
            small Review settings and create accordion
          v-stepper-content(step='4')
            v-card(flat)
              v-card-text
                .subtitle-2.mb-4 Review your accordion configuration:
                v-row
                  v-col(cols='12')
                    v-card(outlined)
                      v-card-text
                        .d-flex.align-center.mb-3
                          v-icon.mr-3(:color='accordion.icon ? "primary" : "grey"') {{ accordion.icon || 'mdi-folder' }}
                          .subtitle-1 {{ accordion.label }}
                          v-chip.ml-3(small, :color='accordion.expanded ? "success" : "default"')
                            | {{ accordion.expanded ? 'Expanded' : 'Collapsed' }}
                        
                        .caption.grey--text.mb-2 STRUCTURE ({{ accordion.children.length }} items):
                        .ml-4
                          .d-flex.align-center.mb-1(v-for='(item, index) in accordion.children', :key='item.id')
                            v-icon.mr-2(small, :color='getItemColor(item.kind)') {{ getItemIcon(item) }}
                            | {{ item.label }}
                            v-chip.ml-2(x-small, :color='getItemColor(item.kind)', text-color='white') {{ item.kind }}
                          .grey--text.caption(v-if='accordion.children.length === 0') No items added
                          
              v-card-actions
                v-btn(text, @click='currentStep = 3') Back
                v-spacer
                v-btn(color='success', @click='createAccordion', :loading='creating')
                  v-icon(left) mdi-check
                  | Create Accordion

    // Bulk Page Selector Dialog
    bulk-page-selector(
      v-model='showBulkPageSelector'
      @select='onBulkPagesSelected'
    )
</template>

<script>
import { v4 as uuid } from 'uuid'
import draggable from 'vuedraggable'
import BulkPageSelector from './bulk-page-selector.vue'

export default {
  name: 'AccordionWizard',
  components: {
    draggable,
    BulkPageSelector
  },
  props: {
    value: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      currentStep: 1,
      creating: false,
      selectedTemplate: null,
      selectedItem: null,
      showBulkPageSelector: false,
      accordion: {
        id: null,
        label: '',
        icon: 'mdi-folder',
        expanded: false,
        children: []
      },
      templates: [
        {
          id: 'blank',
          name: 'Blank',
          description: 'Start with an empty accordion',
          icon: 'mdi-plus',
          color: 'grey',
          items: []
        },
        {
          id: 'documentation',
          name: 'Documentation',
          description: 'Common structure for documentation',
          icon: 'mdi-book-open',
          color: 'blue',
          items: [
            { kind: 'header', label: 'Getting Started', icon: 'mdi-rocket' },
            { kind: 'link', label: 'Installation', icon: 'mdi-download', targetType: 'page', target: '' },
            { kind: 'link', label: 'Quick Start Guide', icon: 'mdi-play', targetType: 'page', target: '' },
            { kind: 'divider', label: '', icon: '' },
            { kind: 'header', label: 'API Reference', icon: 'mdi-code-tags' },
            { kind: 'link', label: 'API Overview', icon: 'mdi-api', targetType: 'page', target: '' },
            { kind: 'link', label: 'Endpoints', icon: 'mdi-web', targetType: 'page', target: '' }
          ]
        },
        {
          id: 'userguide',
          name: 'User Guide',
          description: 'Structure for user manuals',
          icon: 'mdi-account-circle',
          color: 'green',
          items: [
            { kind: 'header', label: 'Overview', icon: 'mdi-eye' },
            { kind: 'link', label: 'Introduction', icon: 'mdi-information', targetType: 'page', target: '' },
            { kind: 'link', label: 'Features', icon: 'mdi-star', targetType: 'page', target: '' },
            { kind: 'divider', label: '', icon: '' },
            { kind: 'header', label: 'How To', icon: 'mdi-help-circle' },
            { kind: 'link', label: 'Basic Usage', icon: 'mdi-play-circle', targetType: 'page', target: '' },
            { kind: 'link', label: 'Advanced Features', icon: 'mdi-cog', targetType: 'page', target: '' }
          ]
        },
        {
          id: 'faq',
          name: 'FAQ',
          description: 'Frequently asked questions',
          icon: 'mdi-help-circle',
          color: 'orange',
          items: [
            { kind: 'header', label: 'General Questions', icon: 'mdi-comment-question' },
            { kind: 'link', label: 'What is this?', icon: 'mdi-help', targetType: 'page', target: '' },
            { kind: 'link', label: 'How do I get started?', icon: 'mdi-rocket-launch', targetType: 'page', target: '' },
            { kind: 'divider', label: '', icon: '' },
            { kind: 'header', label: 'Troubleshooting', icon: 'mdi-wrench' },
            { kind: 'link', label: 'Common Issues', icon: 'mdi-alert-circle', targetType: 'page', target: '' },
            { kind: 'link', label: 'Support', icon: 'mdi-lifebuoy', targetType: 'page', target: '' }
          ]
        }
      ],
      targetTypes: [
        { text: 'Page', value: 'page' },
        { text: 'External URL', value: 'url' },
        { text: 'Anchor', value: 'anchor' }
      ]
    }
  },
  computed: {
    dialog: {
      get() {
        return this.value
      },
      set(val) {
        this.$emit('input', val)
      }
    }
  },
  watch: {
    value(newVal) {
      if (newVal) {
        this.resetWizard()
      }
    }
  },
  methods: {
    resetWizard() {
      this.currentStep = 1
      this.creating = false
      this.selectedTemplate = null
      this.selectedItem = null
      this.showBulkPageSelector = false
      this.accordion = {
        id: uuid(),
        label: '',
        icon: 'mdi-folder',
        expanded: false,
        children: []
      }
    },
    selectTemplate(templateId) {
      this.selectedTemplate = templateId
      const template = this.templates.find(t => t.id === templateId)
      if (template && template.items.length > 0) {
        this.accordion.children = template.items.map(item => ({
          ...item,
          id: uuid()
        }))
      } else {
        this.accordion.children = []
      }
    },
    addWizardItem(kind) {
      const newItem = {
        id: uuid(),
        kind: kind,
        label: this.getDefaultLabel(kind),
        icon: this.getDefaultIcon(kind),
        targetType: kind === 'link' ? 'page' : null,
        target: ''
      }
      this.accordion.children.push(newItem)
      this.selectedItem = newItem
    },
    removeItem(index) {
      if (this.selectedItem && this.accordion.children[index] === this.selectedItem) {
        this.selectedItem = null
      }
      this.accordion.children.splice(index, 1)
    },
    selectItem(item) {
      this.selectedItem = item
    },
    getDefaultLabel(kind) {
      switch (kind) {
        case 'link': return 'New Link'
        case 'header': return 'New Header'
        case 'divider': return ''
        default: return 'New Item'
      }
    },
    getDefaultIcon(kind) {
      switch (kind) {
        case 'link': return 'mdi-link'
        case 'header': return 'mdi-format-title'
        case 'divider': return ''
        default: return 'mdi-circle'
      }
    },
    getItemIcon(item) {
      return item.icon || this.getDefaultIcon(item.kind)
    },
    getItemColor(kind) {
      switch (kind) {
        case 'link': return 'primary'
        case 'header': return 'success'
        case 'divider': return 'grey'
        default: return 'grey'
      }
    },
    capitalizeFirst(str) {
      return str.charAt(0).toUpperCase() + str.slice(1)
    },
    openBulkPageSelector() {
      this.showBulkPageSelector = true
    },
    onBulkPagesSelected(pages) {
      const linkItems = pages.map(page => ({
        id: uuid(),
        kind: 'link',
        label: page.title,
        icon: 'mdi-file-document',
        targetType: 'page',
        target: page.path
      }))
      this.accordion.children.push(...linkItems)
      this.showBulkPageSelector = false
    },
    selectPageForWizardItem() {
      // This would typically open a page selector dialog
      // For now, we'll emit an event that the parent can handle
      this.$emit('select-page', this.selectedItem)
    },
    async createAccordion() {
      this.creating = true
      try {
        // Emit the accordion data to the parent component
        this.$emit('create', this.accordion)
        this.close()
      } catch (err) {
        console.error('Failed to create accordion:', err)
      } finally {
        this.creating = false
      }
    },
    close() {
      this.dialog = false
    }
  }
}
</script>

<style lang="scss" scoped>
.wizard-item {
  border: 1px solid transparent;
  border-radius: 4px;
  margin-bottom: 8px;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    background-color: #f5f5f5;
    border-color: #e0e0e0;
  }

  &.selected {
    background-color: #e3f2fd;
    border-color: #2196f3;
  }
}

.content-builder {
  .sortable-ghost {
    opacity: 0.5;
  }

  .sortable-chosen {
    background-color: #f5f5f5;
  }
}

.item-editor {
  background-color: #fafafa;
}

.orange--border {
  border-color: #ff9800 !important;
}
</style>