<template lang="pug">
  div
    .pa-3.d-flex(v-if='navMode === `MIXED`', :class='$vuetify.theme.dark ? `grey darken-5` : `blue darken-3`')
      v-btn(
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='min-width:0;'
        @click='goHome'
        :aria-label='$t(`common:header.home`)'
        )
        v-icon(size='20') mdi-home
      v-btn.ml-3(
        v-if='currentMode === `custom`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`browse`)'
        )
        v-icon(left) mdi-file-tree
        .body-2.text-none {{$t('common:sidebar.browse')}}
      v-btn.ml-3(
        v-else-if='currentMode === `browse`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`tree`)'
        )
        v-icon(left) mdi-file-tree
        .body-2.text-none {{$t('common:sidebar.tree')}}
      v-btn.ml-3(
        v-else-if='currentMode === `tree`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`custom`)'
        )
        v-icon(left) mdi-navigation
        .body-2.text-none {{$t('common:sidebar.mainMenu')}}
    v-divider
    //-> Custom Navigation
    v-list.py-2(v-if='currentMode === `custom`', dense, :class='color', :dark='dark')
      template(v-for='item of items')
        v-list-item(
          v-if='item.k === `link`'
          :href='item.t'
          :target='item.y === `externalblank` ? `_blank` : `_self`'
          :rel='item.y === `externalblank` ? `noopener` : ``'
          )
          v-list-item-avatar(size='24', tile)
            v-icon(v-if='item.c.match(/fa[a-z] fa-/)', size='19') {{ item.c }}
            v-icon(v-else) {{ item.c }}
          v-list-item-title {{ item.l }}
        v-divider.my-2(v-else-if='item.k === `divider`')
        v-subheader.pl-4(v-else-if='item.k === `header`') {{ item.l }}

    //-> Tree Navigation
    v-treeview(
      v-else-if='currentMode === `tree`'
      activatable
      open-on-click
      :color='"white"'
      :active='treeDefaultActive'
      :open='treeDefaultOpen'
      :items='treeItems'
      :load-children='fetchTreeChild'
      @update:active='activeTreeItem'
    )
      template(v-slot:prepend="{ item, open }")
        v-icon(v-if="!item.children") mdi-text-box
        v-icon(v-else-if="open") mdi-folder-open
        v-icon(v-else) mdi-folder
      template(v-slot:label="{ item }")
        div(class='tree-item')
          a(v-if="!item.children" :href="'/'+item.locale+'/'+item.path")
            span {{item.name}}
          span(v-else) {{item.name}}

    //-> Browse
    v-list.py-2(v-else-if='currentMode === `browse`', dense, :class='color', :dark='dark')
      template(v-if='currentParent.id > 0')
        v-list-item(v-for='(item, idx) of parents', :key='`parent-` + item.id', @click='fetchBrowseItems(item)', style='min-height: 30px;')
          v-list-item-avatar(size='18', :style='`padding-left: ` + (idx * 8) + `px; width: auto; margin: 0 5px 0 0;`')
            v-icon(small) mdi-folder-open
          v-list-item-title {{ item.title }}
        v-divider.mt-2
        v-list-item.mt-2(v-if='currentParent.pageId > 0', :href='`/` + currentParent.locale + `/` + currentParent.path', :key='`directorypage-` + currentParent.id', :input-value='path === currentParent.path')
          v-list-item-avatar(size='24')
            v-icon mdi-text-box
          v-list-item-title {{ currentParent.title }}
        v-subheader.pl-4 {{$t('common:sidebar.currentDirectory')}}
      template(v-for='item of currentItems')
        v-list-item(v-if='item.isFolder', :key='`childfolder-` + item.id', @click='fetchBrowseItems(item)')
          v-list-item-avatar(size='24')
            v-icon mdi-folder
          v-list-item-title {{ item.title }}
        v-list-item(v-else, :href='`/` + item.locale + `/` + item.path', :key='`childpage-` + item.id', :input-value='path === item.path')
          v-list-item-avatar(size='24')
            v-icon mdi-text-box
          v-list-item-title {{ item.title }}
</template>

<script>
import _ from 'lodash'
import gql from 'graphql-tag'
import { get } from 'vuex-pathify'

/* global siteLangs */

export default {
  props: {
    color: {
      type: String,
      default: 'primary'
    },
    dark: {
      type: Boolean,
      default: true
    },
    items: {
      type: Array,
      default: () => []
    },
    navMode: {
      type: String,
      default: 'MIXED'
    }
  },
  data() {
    return {
      currentMode: 'custom',
      currentItems: [],
      currentParent: {
        id: 0,
        title: '/ (root)'
      },
      parents: [],
      loadedCache: [],
      treeItems: [],
      treeDefaultOpen: [],
      treeDefaultActive: [],
    }
  },
  computed: {
    path: get('page/path'),
    locale: get('page/locale')
  },
  methods: {
    switchMode (mode) {
      this.currentMode = mode
      window.localStorage.setItem('navPref', mode)
      if (mode === `browse` && this.loadedCache.length < 1) {
        this.loadFromCurrentPath()
      }
      if (mode === 'tree') {
        this.fetchTreeRoot();
      }
    },
    async fetchBrowseItems (item) {
      this.$store.commit(`loadingStart`, 'browse-load')
      if (!item) {
        item = this.currentParent
      }

      if (this.loadedCache.indexOf(item.id) < 0) {
        this.currentItems = []
      }

      if (item.id === 0) {
        this.parents = []
      } else {
        const flushRightIndex = _.findIndex(this.parents, ['id', item.id])
        if (flushRightIndex >= 0) {
          this.parents = _.take(this.parents, flushRightIndex)
        }
        if (this.parents.length < 1) {
          this.parents.push(this.currentParent)
        }
        this.parents.push(item)
      }

      this.currentParent = item

      const resp = await this.$apollo.query({
        query: gql`
          query ($parent: Int, $locale: String!) {
            pages {
              tree(parent: $parent, mode: ALL, locale: $locale) {
                id
                path
                title
                isFolder
                pageId
                parent
                locale
              }
            }
          }
        `,
        fetchPolicy: 'cache-first',
        variables: {
          parent: item.id,
          locale: this.locale
        }
      })
      this.loadedCache = _.union(this.loadedCache, [item.id])
      this.currentItems = _.get(resp, 'data.pages.tree', [])
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    async loadFromCurrentPath() {
      this.$store.commit(`loadingStart`, 'browse-load')
      const resp = await this.$apollo.query({
        query: gql`
          query ($path: String, $locale: String!) {
            pages {
              tree(path: $path, mode: ALL, locale: $locale, includeAncestors: true) {
                id
                path
                title
                isFolder
                pageId
                parent
                locale
              }
            }
          }
        `,
        fetchPolicy: 'cache-first',
        variables: {
          path: this.path,
          locale: this.locale
        }
      })
      const items = _.get(resp, 'data.pages.tree', [])
      const curPage = _.find(items, ['pageId', this.$store.get('page/id')])
      if (!curPage) {
        console.warn('Could not find current page in page tree listing!')
        return
      }

      let curParentId = curPage.parent
      let invertedAncestors = []
      while (curParentId) {
        const curParent = _.find(items, ['id', curParentId])
        if (!curParent) {
          break
        }
        invertedAncestors.push(curParent)
        curParentId = curParent.parent
      }

      this.parents = [this.currentParent, ...invertedAncestors.reverse()]
      this.currentParent = _.last(this.parents)

      this.loadedCache = [curPage.parent]
      this.currentItems = _.filter(items, ['parent', curPage.parent])
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    goHome () {
      window.location.assign(siteLangs.length > 0 ? `/${this.locale}/home` : '/')
    },
    pageItem2TreeItem(item,level) {
      if (item.isFolder) {
        return { id: item.id, level: level, pageId: item.pageId, path: item.path, locale: item.locale, name: item.title, children: [] }
      } else {
        return { id: item.id, level: level, path: item.path, locale: item.locale, name: item.title }
      }
    },
    activeTreeItem(id) {
      const find = (items) => {
        for(const item of items) {
          if(item.id == id) {
            return item
          }
          if(item.children && item.children.length) {
            const v = find(item.children)
            if(v) {
              return v
            }
          }
        }
      }
      const item = find(this.treeItems)
      if(item) {
        if(!this.treeDefaultActive.includes(item.id)) {
          location.href = `/${item.locale}/${item.path}`
        } else {
          setTimeout(() => {
            const el = document.querySelector(".v-treeview-node--active")
            el.scrollIntoViewIfNeeded()
          })
        }
      }
    },
    async fetchTreeChild(parent) {
      const items = await this.fetchPages(parent.id)
      parent.children = []
      if(parent.pageId){
        parent.children.push({
          id: parent.pageId,level: parent.level+1, path: parent.path, locale: parent.locale, name: parent.name
        })
      }
      parent.children.push(
        ...items.map(item => this.pageItem2TreeItem(item, parent.level+1))
      )
      this.checkTreeDefaultOpen(parent.children);
    },
    async fetchTreeRoot(){
      const children = await this.fetchPages(0)
      this.treeItems = children.map(item => this.pageItem2TreeItem(item, 0))
      this.checkTreeDefaultOpen(this.treeItems, 0);
    },
    async checkTreeDefaultOpen(items){
      const item = items.find(item => item.children && this.path.startsWith(item.path))
      if(item) {
        setTimeout(()=>{
          this.treeDefaultOpen.push(item.id)
        })
      }
      const active = items.find(item => item.path == this.path)
      if(active) {
        this.treeDefaultActive.push(active.id)
      }
    },
    async fetchPages(id) {
      const resp = await this.$apollo.query({
        query: gql`
          query($parent: Int, $locale: String!) {
            pages {
              tree(parent: $parent, mode: ALL, locale: $locale) {
                id
                path
                title
                isFolder
                pageId
                parent
                locale
              }
            }
          }
        `,
        fetchPolicy: 'cache-first',
        variables: {
          parent: id,
          locale: this.locale
        }
      })
      return _.get(resp, 'data.pages.tree', [])
    },
  },
  mounted () {
    this.currentParent.title = `/ ${this.$t('common:sidebar.root')}`
    if (this.navMode === 'TREE') {
      this.currentMode = 'browse'
    } else if (this.navMode === 'STATIC') {
      this.currentMode = 'custom'
    } else if (this.navMode === 'NEWTREE') {
      this.currentMode = 'tree'
    } else {
      this.currentMode = window.localStorage.getItem('navPref') || 'custom'
    }
    if (this.currentMode === 'browse') {
      this.loadFromCurrentPath()
    }
    if (this.currentMode === "tree") {
      this.fetchTreeRoot();
    }
  }
}
</script>


<style lang="scss">
.v-treeview{
  .tree-item {
    font-weight: 500;
    line-height: 1rem;
    font-size: 0.8rem;
  }
  a {
    text-decoration: none;
  }
  &.theme--dark{
    a {
      color: white;
    }
  }
}
</style>
