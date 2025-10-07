<template lang="pug">
  v-dialog(v-model='dialog', max-width='800px', persistent)
    v-card(style='height: 600px;')
      v-toolbar(flat, color='primary', dark)
        v-icon.mr-2 mdi-file-multiple
        .subtitle-1 Select Multiple Pages
        v-spacer
        v-btn(icon, @click='close')
          v-icon mdi-close
          
      v-card-text.pa-0(style='height: calc(100% - 128px);')
        v-row(no-gutters, style='height: 100%;')
          // Page Tree
          v-col(cols='8', style='height: 100%; border-right: 1px solid #e0e0e0; display: flex; flex-direction: column;')
            .d-flex.align-center.px-3.py-2(style='background-color: #f5f5f5; border-bottom: 1px solid #e0e0e0; flex-shrink: 0;')
              v-text-field(
                v-model='searchQuery'
                placeholder='Search pages...'
                prepend-inner-icon='mdi-magnify'
                hide-details
                dense
                clearable
                style='max-width: 300px;'
              )
              v-spacer
              v-chip.mr-2(small, :color='selectedPages.length > 0 ? "success" : "default"') 
                | {{ selectedPages.length }} selected
              v-btn(
                v-if='selectedPages.length > 0'
                small
                text
                color='error'
                @click='clearSelection'
              ) Clear All
                
            .page-tree-container(style='flex: 1; overflow-y: auto;')
              v-treeview(
                v-model='selectedPageIds'
                :items='filteredPageTree'
                :search='searchQuery'
                selectable
                return-object
                dense
                open-all
                selection-type='independent'
                @input='onPageSelection'
              )
                template(v-slot:label='{ item }')
                  .d-flex.align-center
                    v-icon.mr-2(small, :color='getPageIcon(item).color') {{ getPageIcon(item).icon }}
                    span {{ item.name }}
                    v-chip.ml-2(
                      v-if='item.tags && item.tags.length > 0'
                      x-small
                      color='blue'
                      text-color='white'
                    ) {{ item.tags[0] }}
                      
          // Selected Pages Preview
          v-col(cols='4', style='height: 100%; display: flex; flex-direction: column;')
            .d-flex.align-center.px-3.py-2(style='background-color: #f5f5f5; border-bottom: 1px solid #e0e0e0; flex-shrink: 0;')
              v-icon.mr-2 mdi-playlist-check
              .subtitle-2 Selected Pages
              
            .selected-pages-container.pa-3(style='height: calc(100% - 60px); overflow-y: auto;')
              .text-center.py-8.grey--text(v-if='selectedPages.length === 0')
                v-icon(large, color='grey lighten-1') mdi-playlist-plus
                .mt-2 No pages selected
                .caption Select pages from the tree
                
              v-list(v-else, dense)
                v-list-item(
                  v-for='(page, index) in selectedPages'
                  :key='page.id'
                )
                  v-list-item-icon
                    v-icon(small, :color='getPageIcon(page).color') {{ getPageIcon(page).icon }}
                  v-list-item-content
                    v-list-item-title.caption {{ page.name }}
                    v-list-item-subtitle.caption {{ page.path }}
                  v-list-item-action
                    v-btn(icon, x-small, @click='removeFromSelection(index)')
                      v-icon(small) mdi-close
                      
      v-card-actions.px-4.py-3
        .caption.grey--text {{ selectedPages.length }} page(s) will be added as links
        v-spacer
        v-btn(text, @click='close') Cancel
        v-btn(
          color='primary'
          :disabled='selectedPages.length === 0'
          @click='addSelectedPages'
        ) Add {{ selectedPages.length }} Page(s)
</template>

<script>
import gql from 'graphql-tag'

export default {
  name: 'BulkPageSelector',
  props: {
    value: {
      type: Boolean,
      default: false
    },
    currentLocale: {
      type: String,
      default: 'en'
    }
  },
  data() {
    return {
      dialog: false,
      searchQuery: '',
      selectedPageIds: [],
      selectedPages: [],
      pageTree: [],
      loading: false
    }
  },
  computed: {
    filteredPageTree() {
      if (!this.searchQuery) return this.pageTree
      
      const filterTree = (items) => {
        return items.reduce((filtered, item) => {
          const matchesSearch = item.name.toLowerCase().includes(this.searchQuery.toLowerCase()) ||
                               item.path.toLowerCase().includes(this.searchQuery.toLowerCase())
          
          const filteredChildren = item.children ? filterTree(item.children) : []
          
          if (matchesSearch || filteredChildren.length > 0) {
            filtered.push({
              ...item,
              children: filteredChildren
            })
          }
          
          return filtered
        }, [])
      }
      
      return filterTree(this.pageTree)
    }
  },
  watch: {
    value(newVal) {
      this.dialog = newVal
      if (newVal) {
        this.loadPages()
      }
    },
    
    dialog(newVal) {
      if (!newVal) {
        this.$emit('input', false)
        this.resetDialog()
      }
    }
  },
  methods: {
    async loadPages() {
      this.loading = true
      try {
        // Get all pages using the proper query format
        const response = await this.$apollo.query({
          query: gql`
            query($limit: Int, $orderBy: PageOrderBy, $orderByDirection: PageOrderByDirection, $locale: String) {
              pages {
                list(limit: $limit, orderBy: $orderBy, orderByDirection: $orderByDirection, locale: $locale) {
                  id
                  locale
                  path
                  title
                  description
                  createdAt
                  updatedAt
                  tags
                }
              }
            }
          `,
          variables: {
            limit: 1000,
            orderBy: 'TITLE',
            orderByDirection: 'ASC',
            locale: this.currentLocale
          }
        })
        
        this.pageTree = this.transformPageList(response.data.pages.list)
      } catch (error) {
        console.error('Failed to load pages:', error)
        this.$store.commit('showNotification', {
          message: 'Failed to load pages',
          style: 'error',
          icon: 'alert'
        })
      } finally {
        this.loading = false
      }
    },
    
    transformPageList(items) {
      // Build a hierarchical tree structure from flat page paths
      const pathMap = new Map()
      const roots = []
      
      // First pass: create all items and folders
      items.forEach(item => {
        const pathParts = item.path.split('/').filter(part => part !== '')
        
        // Create folder structure
        for (let i = 0; i < pathParts.length; i++) {
          const currentPath = '/' + pathParts.slice(0, i + 1).join('/')
          
          if (!pathMap.has(currentPath)) {
            const isLeafPage = (i === pathParts.length - 1)
            const folderItem = {
              id: isLeafPage ? item.id : `folder-${currentPath}`,
              name: isLeafPage ? item.title : pathParts[i],
              path: currentPath,
              isFolder: !isLeafPage,
              pageId: isLeafPage ? item.id : null,
              children: [],
              originalPage: isLeafPage ? item : null
            }
            pathMap.set(currentPath, folderItem)
          }
        }
      })
      
      // Second pass: build parent-child relationships
      pathMap.forEach((item, path) => {
        const pathParts = path.split('/').filter(part => part !== '')
        
        if (pathParts.length === 1) {
          // Root level item
          roots.push(item)
        } else {
          // Find parent
          const parentPath = '/' + pathParts.slice(0, -1).join('/')
          const parent = pathMap.get(parentPath)
          if (parent) {
            parent.children.push(item)
          }
        }
      })
      
      // Sort children by name
      const sortChildren = (items) => {
        items.forEach(item => {
          if (item.children.length > 0) {
            item.children.sort((a, b) => {
              // Folders first, then pages
              if (a.isFolder && !b.isFolder) return -1
              if (!a.isFolder && b.isFolder) return 1
              return a.name.localeCompare(b.name)
            })
            sortChildren(item.children)
          }
        })
      }
      
      roots.sort((a, b) => {
        if (a.isFolder && !b.isFolder) return -1
        if (!a.isFolder && b.isFolder) return 1
        return a.name.localeCompare(b.name)
      })
      
      sortChildren(roots)
      return roots
    },
    
    onPageSelection(selectedItems) {
      // Filter out folder items and keep only actual pages
      this.selectedPages = selectedItems.filter(item => !item.isFolder)
    },
    
    getPageIcon(page) {
      if (page.isFolder) {
        return { icon: 'mdi-folder', color: 'orange' }
      }
      
      // Determine icon based on page path/name
      const path = page.path.toLowerCase()
      const name = page.name.toLowerCase()
      
      if (path.includes('/api/') || name.includes('api')) {
        return { icon: 'mdi-api', color: 'green' }
      }
      if (path.includes('/guide/') || name.includes('guide')) {
        return { icon: 'mdi-book-open-page-variant', color: 'blue' }
      }
      if (path.includes('/tutorial/') || name.includes('tutorial')) {
        return { icon: 'mdi-school', color: 'purple' }
      }
      if (path.includes('/reference/') || name.includes('reference')) {
        return { icon: 'mdi-library', color: 'teal' }
      }
      if (path.includes('/faq/') || name.includes('faq')) {
        return { icon: 'mdi-help-circle', color: 'orange' }
      }
      
      return { icon: 'mdi-file-document-outline', color: 'grey' }
    },
    
    removeFromSelection(index) {
      const removedPage = this.selectedPages[index]
      this.selectedPages.splice(index, 1)
      
      // Also remove from selectedPageIds
      const idIndex = this.selectedPageIds.findIndex(item => item.id === removedPage.id)
      if (idIndex !== -1) {
        this.selectedPageIds.splice(idIndex, 1)
      }
    },
    
    clearSelection() {
      this.selectedPages = []
      this.selectedPageIds = []
    },
    
    addSelectedPages() {
      const pagesAsLinks = this.selectedPages.map(page => ({
        id: `temp_${Date.now()}_${Math.random()}`,
        kind: 'link',
        label: page.name,
        icon: 'mdi-file-document-outline',
        targetType: 'page',
        target: `/${this.currentLocale}${page.path}`,
        visibilityMode: 'all',
        visibilityGroups: []
      }))
      
      this.$emit('pages-selected', pagesAsLinks)
      this.close()
    },
    
    close() {
      this.dialog = false
    },
    
    resetDialog() {
      this.searchQuery = ''
      this.selectedPageIds = []
      this.selectedPages = []
    }
  }
}
</script>

<style lang="scss" scoped>
.page-tree-container {
  background-color: #fafafa;
}

.selected-pages-container {
  background-color: #f8f9fa;
}

::v-deep .v-treeview-node__label {
  font-size: 13px;
}

::v-deep .v-treeview-node__content {
  margin-left: 0;
}

::v-deep .v-list-item__content {
  padding: 4px 0;
}
</style>