<template>
  <div :class="[theme.datatable, staticClass]">
    <div :class="theme.datatable__wrapper" data-elm="wrapper">
      <data-table-filter v-if="!disableFiltering" :filter.sync="options.filter" />
      <data-table-pagination
        v-if="paginationTop" :per-page="perPage" :page.sync="p"
        :total="total" :each-side="paginationSide"
      />
      <div :class="theme.datatable__screen" data-elm="screen">
        <loading v-if="!disableLoader" :active="loading">
          <slot name="loader" />
        </loading>
        <table
          :class="theme.datatable__content" cellspacing="0" cellpadding="0"
          data-elm="content"
        >
          <data-table-head :columns="columns" :sort-by.sync="options.sortBy" :sort-desc.sync="options.sortDesc" />
          <data-table-body
            :columns="columns" :items="actualItems"
            :click="rowClick" :hover="rowHover"
          />
        </table>
      </div>
      <data-table-pagination
        v-if="paginationBottom" :per-page="perPage" :page.sync="p"
        :total="total" :each-side="paginationSide"
      />
    </div>
  </div>
</template>
<script>
import { defaultProps, i18nMixin } from './helpers'
import { load, transform, themeDefault } from 'teible'
import clone from 'clone'
import DataTableBody from './DataTableBody.vue'
import DataTableHead from './DataTableHead.vue'
import DataTablePagination from './DataTablePagination.vue'
import DataTableFilter from './DataTableFilter.vue'
import Loading from './Loading.vue'

export default {
  name: 'DataTable',
  components: { DataTableBody, DataTableHead, DataTablePagination, DataTableFilter, Loading },
  mixins: [i18nMixin],
  props: {
    items: {
      type: [Array, Function],
      required: true
    },
    page: {
      type: Number,
      default: 1
    },
    perPage: {
      type: Number,
      default: 10
    },
    sortBy: {
      type: String,
      default: ''
    },
    sortDesc: {
      type: Boolean,
      default: false
    },
    filter: {
      type: String,
      default: ''
    },
    theme: {
      type: Object,
      default () {
        return themeDefault
      }
    },
    disableFiltering: {
      type: Boolean,
      default: false
    },
    disableLoader: {
      type: Boolean,
      default: false
    },
    pagination: {
      type: Array,
      default () {
        return ['bottom']
      }
    },
    paginationSide: {
      type: Number,
      default: 2
    },
    rowClick: {
      type: Function,
      default: (event, item, index) => {}
    },
    rowHover: {
      type: Function,
      default: (event, item, index) => {}
    }
  },
  provide () {
    return {
      $theme: () => this.theme // because provide & inject bindings are not reactive
    }
  },
  data () {
    return {
      actualItems: [],
      vnodes: [],
      total: 0,
      p: this.page,
      options: {
        sortBy: this.sortBy,
        sortDesc: this.sortDesc,
        filter: this.filter
      },
      loading: false,
      staticClass: ''
    }
  },
  computed: {
    paginationTop () {
      return this.pagination.indexOf('top') > -1
    },
    paginationBottom () {
      return this.pagination.indexOf('bottom') > -1
    },
    func () {
      return this.items instanceof Function
    },
    identifier () {
      if (this.disableFiltering) {
        return `by:${this.sorting.by}|order:${this.sorting.order}|page:${this.p}|per_page:${this.perPage}`
      }

      return `by:${this.sorting.by}|order:${this.sorting.order}|filter:${this.options.filter}|page:${this.p}|per_page:${this.perPage}`
    },
    columns () {
      return this.vnodes.map(vnode => {
        const { componentOptions: { Ctor: { options: { props } }, propsData, children }, data: { scopedSlots, attrs, class: dynamicClass, staticClass } } = vnode
        const { field, label, sortable, filterable, render } = defaultProps(props, propsData)
        return {
          field,
          label,
          sortable,
          filterable,
          render,
          scopedSlots,
          children,
          attrs,
          dynamicClass,
          staticClass
        }
      })
    },
    filterable () {
      return this.columns
        .filter(column => {
          return column.filterable
        })
        .map(column => {
          return column.field
        })
        .filter(field => field)
    },
    filtering () {
      if (this.disableFiltering) {
        // Data should not be filtered without query
        return {
          query: ''
        }
      }

      return {
        query: this.options.filter.toLowerCase(),
        fields: this.filterable
      }
    },
    paging () {
      return {
        page: this.p,
        perPage: this.perPage
      }
    },
    sorting () {
      return {
        by: this.options.sortBy,
        order: !this.options.sortDesc ? 'asc' : 'desc'
      }
    },
    transformed () {
      if (this.func) {
        return []
      }

      return this.transform(clone(this.items, false))
    }
  },
  watch: {
    items: 'loadItems',
    identifier: 'loadItems',
    sortBy: {
      immediate: true,
      handler (val) {
        this.$set(this.options, 'sortBy', val)
      }
    },
    sortDesc: {
      immediate: true,
      handler (val) {
        this.$set(this.options, 'sortDesc', val)
      }
    },
    filter: {
      immediate: true,
      handler (val) {
        this.$set(this.options, 'filter', val)
        this.p = 1
      }
    },
    'options.sortBy' (val) {
      this.$emit('update:sortBy', val)
    },
    'options.sortDesc' (val) {
      this.$emit('update:sortDesc', val)
    },
    'options.filter' (val) {
      this.$emit('update:filter', val)
    },
    p (val) {
      this.$emit('update:page', val)
    },
    page: {
      immediate: true,
      handler  (val) {
        if (val === this.p) {
          return
        }

        this.p = val
      }
    }
  },
  created () {
    this.loadClass()
    this.loadSlots()
    this.loadItems()
  },
  methods: {
    transform (data) {
      return transform(data, this.columns)
    },
    loadSlots () {
      // $slots is not reactive
      this.vnodes = !this.$slots.default ? [] : this.$slots.default.filter(vnode => vnode.componentOptions)
    },
    loadClass () {
      this.staticClass = this.$vnode.data.staticClass
    },
    loadItems () {
      if (this.func) {
        this.loading = true
        Promise.resolve(this.items(this.filtering, this.sorting, this.paging))
          .then(data => {
            this.actualItems = this.transform(data.items)
            this.total = data.total
          })
          .then(() => {
            this.loading = false
          })
          .catch(() => {
            this.loading = false
          })
          .finally(this.ping)

        return
      }

      if (!this.items) {
        this.actualItems = []
        this.total = 0
        return this.ping()
      }

      const data = load(this.transformed, this.filtering, this.sorting, this.paging)
      this.actualItems = data.items
      this.total = data.total
      return this.ping()
    },
    reloadItems () {
      if (this.p === 1) {
        return this.loadItems()
      }

      this.p = 1 // it will trigger loadItems internally
    },
    ping () {
      this.$emit('loaded', {
        items: this.actualItems,
        total: this.total
      })
    }
  }
}
</script>
