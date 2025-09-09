● Wiki.js Accordion Navigation Implementation Summary

  Problem

  Wiki.js navigation sidebar needed accordion functionality to organize menu items hierarchically, but
  Vuetify's v-list-group caused parent headers to disappear when expanded.

  Solution Overview

  Created a fully functional accordion navigation system with proper nesting support by consolidating all logic    
   into a single self-contained component.

  Key Files Modified

  1. GraphQL Schema (server/graph/schemas/navigation.graphql)

  # Added accordion support to NavigationItem and NavigationItemInput
  expanded: Boolean
  children: [NavigationItem] # or [NavigationItemInput] for input

  2. Navigation Model (server/models/navigation.js)

  // Added recursive authorization filtering for children
  static getAuthorizedItems(tree = [], groups = []) {
    return _.filter(tree, leaf => {
      const isAuthorized = leaf.visibilityMode === 'all' || _.intersection(leaf.visibilityGroups,
  groups).length > 0
      if (isAuthorized && leaf.children && leaf.children.length > 0) {
        leaf.children = WIKI.models.navigation.getAuthorizedItems(leaf.children, groups)
      }
      return isAuthorized
    })
  }

  3. Server Data Transformation (server/controllers/common.js)

  // Added recursive transformation for sidebar data
  function transformNavItem(n) {
    const transformed = {
      i: `sdi-${sdi++}`, k: n.kind, l: n.label, c: n.icon, y: n.targetType, t: n.target
    }
    if (n.kind === 'accordion') {
      transformed.expanded = n.expanded
      transformed.children = n.children ? n.children.map(transformNavItem) : []
    }
    return transformed
  }

  4. Admin Interface (client/components/admin/admin-navigation.vue)

  // Added accordion to dropdown menu
  v-list-item(@click='addItem("accordion")')
    v-list-item-avatar(size='24'): v-icon mdi-folder
    v-list-item-title Accordion

  // Added accordion case to addItem method
  case 'accordion':
    newItem = { ...newItem, label: 'New Accordion', icon: 'mdi-folder', expanded: false, children: [] }

  // Added child management UI with proper selection/editing

  5. Accordion Component (client/themes/default/components/nav-accordion-item.vue)

  <!-- Self-contained component handling ALL navigation item types -->
  <template lang="pug">
    div
      v-list-item(v-if='item.k === `link`' ...) // Link items
      v-divider(v-else-if='item.k === `divider`' ...) // Dividers
      v-subheader(v-else-if='item.k === `header`' ...) // Headers
      template(v-else-if='item.k === `accordion`')
        v-list-item(@click='toggleExpanded' ...)
          v-list-item-avatar: v-icon {{ item.c || 'mdi-folder' }}
          v-list-item-title {{ item.l }}
          v-list-item-action: v-icon {{ isExpanded ? 'mdi-chevron-up' : 'mdi-chevron-down' }}
        nav-accordion-item(v-if='isExpanded' v-for='subItem of item.children' :item='subItem' :level='level +      
  1')
  </template>

  <script>
  export default {
    data: () => ({ internalExpanded: false }),
    computed: { isExpanded() { return this.item.k === 'accordion' ? this.internalExpanded : false }},
    methods: { toggleExpanded() { this.internalExpanded = !this.internalExpanded }},
    mounted() { if (this.item.k === 'accordion') this.internalExpanded = this.item.expanded !== false }
  }
  </script>

  6. Main Sidebar (client/themes/default/components/nav-sidebar.vue)

  <!-- Simplified to use ONLY nav-accordion-item for everything -->
  v-list.py-2(v-if='currentMode === `custom`' ...)
    nav-accordion-item(v-for='(item, index) of items' :item='item' :level='0')

  Usage Workflow

  1. Admin: Navigation → Mixed/Static mode → Add → Accordion
  2. Configure: Set label, icon, expanded state
  3. Add Children: Click "Add Child Item" → Choose type (link/header/divider/nested accordion)
  4. Edit Children: Click any child item to select and edit (URLs, page selection, etc.)
  5. Save: Click Apply
  6. Result: Sidebar shows accordion with parent always visible, clickable toggle, proper nesting

  Key Features

  - ✅ Parent accordion header ALWAYS visible (never disappears)
  - ✅ Unlimited nesting levels (accordion within accordion)
  - ✅ Self-contained component logic (no mixed rendering)
  - ✅ Proper state management with reactive toggling
  - ✅ Full admin interface with child item management
  - ✅ Recursive data transformation and authorization
  - ✅ Clean expand/collapse with animated chevrons
  - ✅ Proper indentation at all nesting levels

  Critical Fix

  Never use Vuetify's v-list-group - it replaces activator content when expanded. Use custom template with
  v-list-item + conditional children rendering instead.