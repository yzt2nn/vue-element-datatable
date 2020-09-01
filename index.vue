<template>
  <div class="datatable">
    <el-button
      v-if="insertButton"
      class="insert-begin-button"
      type="primary"
      @click="handleBeginInsert"
    >{{ insertButton }}</el-button>
    <el-input
      v-if="search"
      v-model="searchKw"
      :placeholder="search === true ? '输入关键字搜索' : search"
      class="search-bar"
      size="mini"
      clearable
      @change="handleSearchChange"
    />
    <el-table
      ref="table"
      v-loading="isLoading"
      :data="viewList"
      :border="border"
      :stripe="stripe"
      :class="search && !insertButton ? 'has-search-bar': ''"
      style="width: 100%"
      @sort-change="handleSortChange"
      @row-click="handleRowClick"
      @selection-change="handleSelectionChange"
    >
      <el-table-column v-if="onSelectionChange" :selectable="selectable" type="selection" width="55" />
      <el-table-column
        v-for="(header, index) in tableHeaders"
        :key="index"
        :prop="header.prop"
        :label="header.label"
        :sortable="isSortable || header.sortable"
        :sort-orders="['ascending', 'descending']"
        :width="header.width ? header.width : false"
      >
        <template slot-scope="scope">
          <slot :name="header.prop" :row="scope.row" :$index="scope.$index">
            <div v-if="scope.row && scope.row.__edit">
              <el-input v-model="scope.row[header.prop]" size="mini" />
              <div
                v-if="scope.row.__validateMsgs[header.prop]"
                class="error-message"
              >{{ scope.row.__validateMsgs[header.prop] }}</div>
            </div>
            <span
              v-else
            >{{ scope.row ? ( header.formatter ? header.formatter(scope.row[header.prop]) : scope.row[header.prop]): '' }}</span>
          </slot>
        </template>
      </el-table-column>
      <el-table-column
        v-if="editButton || deleteButton || insertCount > 0 || hasOperations"
        :label="tipMsgs.operation"
        :width="operationWidth"
      >
        <template slot-scope="scope">
          <el-button
            v-if="editButton && scope.row && !scope.row.__edit"
            size="mini"
            @click="handleEdit(scope.$index, scope.row)"
          >{{ editButton }}</el-button>
          <el-button
            v-if="scope.row && scope.row.__insert"
            :loading="insertLoading"
            size="mini"
            type="primary"
            @click="handleCommitInsert(scope.$index, scope.row)"
          >添加</el-button>
          <el-button
            v-else-if="scope.row && scope.row.__edit"
            :loading="scope.row.__editLoading"
            size="mini"
            type="primary"
            @click="handleCommitEdit(scope.$index, scope.row)"
          >确定</el-button>
          <el-button
            v-if="deleteButton && scope.row && !scope.row.__edit"
            :loading="scope.row.__deleteLoading"
            size="mini"
            type="danger"
            @click="handleDelete(scope.$index, scope.row)"
          >{{ deleteButton }}</el-button>
          <el-button
            v-if="scope.row && scope.row.__edit"
            size="mini"
            @click="handleCancelEdit(scope.$index, scope.row)"
          >取消</el-button>
          <slot :row="scope.row" :$index="scope.$index" name="_operation" />
        </template>
      </el-table-column>
    </el-table>
    <div class="bottom-bar">
      <slot name="leftBottom" />
      <el-pagination
        v-if="pagination"
        ref="pagination"
        :disabled="isLoading"
        :current-page="currentPage"
        :page-size="size"
        :page-sizes="sizes"
        :total="total"
        :layout="paginationLayout ? paginationLayout : defaultPaginationLayout"
        background
        @size-change="handleSizeChange"
        @current-change="handlePageChange"
      />
    </div>
  </div>
</template>
<script>
import axios from 'axios'

export default {
  props: {
    init: {
      type: Boolean,
      default: true
    },
    reloadPrefix: {
      type: String,
      default: ''
    },
    tableHeaders: {
      // 表头
      type: Array,
      default: () => {
        return []
      }
    },
    tableData: {
      // 表格总数据
      type: Array,
      default: () => {
        return []
      }
    },
    page: {
      type: Number,
      default: 1
    },
    totalCount: {
      type: Number,
      default: undefined
    },
    pageSize: {
      type: Number,
      default: 10
    },
    pageSizes: {
      type: Array,
      default: () => {
        return [5, 10, 20, 50, 100]
      }
    },
    showPageSizeSelector: {
      type: Boolean,
      default: true
    },
    showTotalCount: {
      type: Boolean,
      default: true
    },
    showPageJumper: {
      type: Boolean,
      default: true
    },
    showPageSelector: {
      type: Boolean,
      default: true
    },
    pagination: {
      type: Boolean,
      default: true
    },
    paginationLayout: {
      type: String,
      default: ''
    },
    sortable: {
      type: [String, Boolean],
      default: false
    },
    stripe: {
      type: Boolean,
      default: true
    },
    border: {
      type: Boolean,
      default: true
    },
    align: {
      type: String,
      default: 'left'
    },
    insertButton: {
      type: String,
      default: ''
    },
    editButton: {
      type: String,
      default: ''
    },
    deleteButton: {
      type: String,
      default: ''
    },
    search: {
      type: [String, Boolean],
      default: ''
    },
    url: {
      type: String,
      default: ''
    },
    resObjName: {
      type: String,
      default: ''
    },
    dataName: {
      type: String,
      default: 'data'
    },
    countName: {
      type: String,
      default: 'total'
    },
    statusName: {
      type: String,
      default: 'status'
    },
    insertIdName: {
      type: String,
      default: 'id'
    },
    keywordName: {
      type: String,
      default: 'keyword'
    },
    orderPropName: {
      type: String,
      default: 'prop'
    },
    orderTypeName: {
      type: String,
      default: 'order'
    },
    pageName: {
      type: String,
      default: 'page'
    },
    sizeName: {
      type: String,
      default: 'size'
    },
    successMark: {
      type: String,
      default: 'ok'
    },
    type: {
      type: String,
      default: 'frontend'
    },
    requestInstance: {
      type: [Object, Function],
      default: null
    },
    tipMessages: {
      type: Object,
      default: () => {
        return {}
      }
    },
    operationWidth: {
      type: [Number, Boolean],
      default: false
    },
    onCommitInsert: {
      type: Function,
      default: null
    },
    onCommitEdit: {
      type: Function,
      default: null
    },
    onCommitDelete: {
      type: Function,
      default: null
    },
    onSizeChange: {
      type: Function,
      default: null
    },
    onSearchChange: {
      type: Function,
      default: null
    },
    onSortChange: {
      type: Function,
      default: null
    },
    onPageChange: {
      type: Function,
      default: null
    },
    onSelectionChange: {
      type: Function,
      default: null
    },
    onRowClick: {
      type: Function,
      default: null
    },
    extendParams: {
      type: Object,
      default: null
    },
    preprocess: {
      type: Function,
      default: null
    },
    selectable: {
      type: Function,
      default: null
    },
    onDataLoaded: {
      type: Function,
      default: null
    }
  },
  data() {
    return {
      currentPage: 1,
      oldPage: 0,
      total: null,
      viewList: [], // 表格显示的部分（page size）
      isLoading: true,
      size: 0,
      sizes: [],
      searchKw: '',
      insertCount: 0,
      filterData: [],
      oTableData: [],
      sortInfo: {},
      insertLoading: false,
      isFrontendLoaded: false,
      isSilenceLoading: false,
      tipMsgs: {
        operation: '操作',
        get: { success: '查询成功！', fail: '查询失败！' },
        insert: { success: '新增条目成功！', fail: '新增条目失败！' },
        update: { success: '更新成功！', fail: '更新失败！' },
        delete: { success: '删除成功！', fail: '删除失败！' },
        deleteConfirm: {
          title: '提示',
          tip: '此操作将永久删除该条目, 是否继续?',
          ok: '继续',
          cancel: '取消'
        },
        page: { success: '查询成功！', fail: '查询失败！' },
        size: { success: '查询成功！', fail: '查询失败！' },
        search: { success: '查询成功！', fail: '查询失败！' },
        sort: { success: '查询成功！', fail: '排序查询失败！' },
        warn: {
          inserting: '有未提交的新数据！',
          waiting: '正在进行操作，请稍后！'
        }
      },
      currentParams: null
    }
  },
  computed: {
    isSortable() {
      return this.insertCount > 0 ? false : this.sortable
    },
    reqIns() {
      if (this.requestInstance) return this.requestInstance
      else return this.$http ? this.$http : axios.create()
    },
    hasOperations() {
      if (
        (this.$slots && this.$slots._operation) ||
        (this.$scopedSlots && this.$scopedSlots._operation)
      ) {
        return true
      }
      return false
    },
    defaultPaginationLayout() {
      let layoutStr = `${this.showTotalCount ? 'total,' : ''}${
        this.showPageSizeSelector ? 'sizes,' : ''
      }${this.showPageSelector ? 'prev,pager,next,' : ''}${
        this.showPageJumper ? 'jumper,' : ''
      }`
      layoutStr = layoutStr.substring(0, layoutStr.lastIndexOf(','))
      return layoutStr
    }
  },
  watch: {
    currentPage: function(newVal, oldVal) {
      if (this.type === 'frontend') this.calcViewList()
    },
    page(val) {
      this.currentPage = val
    },
    pageSize() {
      // 只有前端分页时，才在这里计算viewlist
      // 后端分页是在父组件中直接改变tabledata
      this.size = this.pageSize
      if (this.type === 'frontend') this.calcViewList()
    },
    totalCount(val) {
      if (this.type === 'backend') this.calcTotalCount()
    },
    tableData: {
      handler(newVal, oldVal) {
        if (!newVal) return
        if (this.size === 0) {
          this.calcPageSizes()
        }
        if (newVal !== oldVal) {
          setTimeout(() => {
            this.initTableData()
            this.calcTotalCount()
            this.calcViewList()
          }, 0)
        }
      }
      // immediate: true
    },
    extendParams(newVal) {
      if (newVal[this.sizeName]) this.size = Number(newVal[this.sizeName])
      delete newVal[this.sizeName]
      this.isFrontendLoaded = false
      if (newVal[this.pageName]) {
        this.handlePageChange(Number(newVal[this.pageName]))
        delete newVal[this.pageName]
      } else this.handlePageChange(1)
    },
    tipMessages(newVal) {
      this.myExtend(this.tipMsgs, newVal, true)
    }
  },
  mounted() {
    this.myExtend(this.tipMsgs, this.tipMessages, true)
    this.calcPageSizes()
    if (this.url) {
      if (
        this.reloadPrefix &&
        sessionStorage.getItem(this.reloadPrefix + '-dt-reload') &&
        sessionStorage.getItem(this.reloadPrefix + '-dt-reload') === 'false' &&
        sessionStorage.getItem(this.reloadPrefix + '-params')
      ) {
        sessionStorage.removeItem(this.reloadPrefix + '-dt-reload')
      } else {
        if (this.reloadPrefix) {
          sessionStorage.removeItem(this.reloadPrefix + '-dt-reload')
        }
        this.handlePageChange(1)
      }
    }
  },
  methods: {
    reload(holdPage, isSilence, deleting) {
      this.isFrontendLoaded = false
      this.isSilenceLoading = !!isSilence
      let offset = 0
      if (deleting && this.viewList.length <= 1) {
        offset = 1
      }
      this.handlePageChange(holdPage ? this.currentPage - offset || 1 : 1)
    },
    calcPageSizes() {
      this.size = this.pageSize
      this.sizes = this.pageSizes

      if (this.sizes.length === 0) {
        this.sizes = [this.size]
      } else {
        if (this.sizes.indexOf(this.size) === -1) {
          // 如果给定的预设size没在sizes中，则按数字大小插入sizes
          for (let i = 0, length = this.sizes.length; i < length; i++) {
            if (this.sizes[i] > this.size) {
              this.sizes.splice(i, 0, this.size)
              break
            }
          }
        }
      }
    },
    getParams() {
      return this.currentParams
    },
    myExtend(a, b, deep) {
      for (const prop in b) {
        if (a[prop]) {
          // 若存在
          if (
            deep &&
            typeof b[prop] === 'object' &&
            typeof a[prop] === 'object'
          ) {
            this.myExtend(a[prop], b[prop], deep)
          } else {
            a[prop] = b[prop]
          }
        } else {
          a[prop] = b[prop]
        }
      }
      return a
    },
    popItem(arr, item) {
      const i = arr.indexOf(item)
      if (i > -1) {
        return arr.splice(i, 1)
      }
      return -1
    },
    cleanFormData(data) {
      const d = {}
      for (const i in data) {
        d[i] = data[i]
      }
      delete d.oData
      delete d.__edit
      delete d.__insert
      delete d.__deleteLoading
      delete d.__editLoading
      delete d.__validateMsgs

      return d
    },
    doValidate(data) {
      let ok = true
      data.__validateMsgs = {}
      for (const i in this.tableHeaders) {
        let check = false // 在当前字段是否检查出错误
        const propName = this.tableHeaders[i].prop
        const d = data[propName]
        const vls = this.tableHeaders[i].validators
        if (vls) {
          // 开始验证
          for (const j in vls) {
            const v = vls[j]
            switch (v.type) {
              case 'required': {
                if (!d) {
                  data.__validateMsgs[propName] = v.message
                  ok = false
                  check = true
                }
                break
              }
              case 'length': {
                if (
                  (v.min && d.length < v.min) ||
                  (v.max && d.length > v.max)
                ) {
                  data.__validateMsgs[propName] = v.message
                  ok = false
                  check = true
                }
                break
              }
              case 'include': {
                if (v.list && v.list.indexOf(d) === -1) {
                  data.__validateMsgs[propName] = v.message
                  ok = false
                  check = true
                }
                break
              }
              case 'regex': {
                if (v.regex && !v.regex.test(d)) {
                  data.__validateMsgs[propName] = v.message
                  ok = false
                  check = true
                }
                break
              }
              case 'custom': {
                if (v.func && !v.func(d)) {
                  data.__validateMsgs[propName] = v.message
                  ok = false
                  check = true
                }
                break
              }
            }
            if (check) break
          }
        }
      }
      return ok
    },
    urlGet(params, success, fail, page) {
      if (this.extendParams) {
        this.myExtend(params, this.extendParams)
      }
      this.currentParams = params
      this.reqIns
        .get(this.url, {
          params
        })
        .then(resp => {
          let resObj = null
          if (this.resObjName) {
            // 设置了对象名的情况
            if (resp[this.resObjName]) {
              // 若resp在上层提取了data
              resObj = resp[this.resObjName]
            } else if (resp['data'] && resp['data'][this.resObjName]) {
              // 没有提取resp中data的情况
              resObj = resp['data'][this.resObjName]
            }
          } else if (resp instanceof Array) {
            // 若resp在上层提取了data，且第一级就是数组的情况
            resObj = {}
            resObj[this.dataName] = resp
          } else if (resp[this.dataName] instanceof Array) {
            // 若在resp提取了data，且第一级是对象
            resObj = resp
          } else if (resp['data'] instanceof Array) {
            // 返回值直接为数组的情况
            resObj = {}
            resObj[this.dataName] = resp['data']
          } else if (
            resp['data'] &&
            resp['data'][this.dataName] instanceof Array
          ) {
            // 一般情况，类似resp: {data:{data:[], total:100}}
            resObj = resp['data']
          }
          if (resObj) {
            let final_data = resObj[this.dataName]
            if (this.preprocess) {
              final_data = this.preprocess(final_data, resp)
            }
            this.setTableData(final_data)
            if (this.type === 'backend' && resObj[this.countName]) {
              this.total = resObj[this.countName]
            } else {
              this.total = this.tableData.length
            }
            if (page) this.currentPage = page
            this.calcViewList()
            this.isFrontendLoaded = true
            if (success) success()
          } else if (fail) fail()
        })
        .catch(() => {
          if (fail) {
            fail()
          }
        })
    },
    urlPost(params, success, fail) {
      this.reqIns
        .post(this.url, {
          params
        })
        .then(resp => {
          if (resp['data'][this.statusName] === this.successMark) {
            if (success) success(resp)
          } else if (fail) fail()
        })
        .catch(() => {
          if (fail) fail()
        })
    },
    urlPut(params, success, fail) {
      this.reqIns
        .put(this.url, {
          params
        })
        .then(resp => {
          if (resp['data'][this.statusName] === this.successMark) {
            if (success) success()
          } else if (fail) fail()
        })
        .catch(() => {
          if (fail) fail()
        })
    },
    urlDelete(params, success, fail) {
      this.reqIns
        .delete(this.url, {
          params
        })
        .then(resp => {
          if (resp['data'][this.statusName] === this.successMark) {
            if (success) success()
          } else if (fail) fail()
        })
        .catch(() => {
          if (fail) fail()
        })
    },
    setTableData(newArr) {
      this.tableData.splice(0, this.tableData.length)
      for (const i in newArr) this.tableData.push(newArr[i])
      this.initTableData()
    },
    showMsg(type, msg, dft, closeLoading) {
      if (closeLoading) this.isLoading = false
      this.$message({
        showClose: true,
        type,
        message: msg || dft
      })
    },
    setOData(data) {
      const oData = {}
      for (const ii in this.tableHeaders) {
        const propName = this.tableHeaders[ii].prop
        oData[propName] = data[propName]
      }
      data.oData = oData
    },
    initTableData() {
      // 添加edit标记、删除中标记等与源数据备份，以便撤回编辑
      for (const i in this.tableData) {
        const data = this.tableData[i]
        this.$set(data, '__edit', false)
        this.$set(data, '__deleteLoading', false)
        this.$set(data, '__editLoading', false)
        this.$set(data, '__validateMsgs', {})
        this.setOData(data)
      }
    },
    calcViewList() {
      if (this.type === 'frontend') {
        // 前端分页
        const from = this.pagination ? ((this.currentPage - 1) * this.size) : 0
        const to = this.pagination ? (from + this.size) : undefined
        if (this.searchKw) {
          this.viewList = this.filterData.slice(from, to)
        } else this.viewList = this.tableData.slice(from, to)
      } else {
        // 后端分页
        this.viewList = this.tableData.slice(0)
      }
      this.isLoading = false
      if (this.onDataLoaded) {
        this.onDataLoaded(this.viewList)
      }
    },
    calcTotalCount() {
      if (this.type === 'backend') this.total = this.totalCount
      else this.total = this.tableData.length
    },
    handlePageChange(val) {
      this.$emit('beforeChangePage', { page: val })
      if (!this.isSilenceLoading) {
        this.isLoading = true
      }
      this.insertCount = 0
      this.currentPage = val
      // this.$refs.pagination.internalCurrentPage = val;

      const params = {}
      params[this.pageName] = val
      params[this.sizeName] = this.size
      params[this.orderTypeName] = this.sortInfo.order
      params[this.orderPropName] = this.sortInfo.prop
      params[this.keywordName] = this.searchKw

      if (this.type === 'backend') {
        if (this.url && !this.onPageChange) {
          // 有url的情况下，组件自行请求数据
          this.urlGet(params, null, null, val)
        } else {
          // 后台分页
          this.onPageChange(
            params,
            () => {
              this.isLoading = false
            },
            msg => {
              this.showMsg('error', msg, this.tipMsgs.page.fail, true)
            }
          )
        }
      } else {
        if (this.url && !this.isFrontendLoaded) this.urlGet(params)
      }
    },
    handleEdit(index, row) {
      row.__edit = true
    },
    deleteSuccessCallback(row, msg) {
      // 从总列表中删除此项
      this.isLoading = true
      this.total = this.total - 1 > 0 ? this.total - 1 : 0
      if (this.total % this.size === 0) {
        const page = Math.ceil(this.total / this.size)
        if (this.currentPage > page) this.currentPage = page
      }
      if (this.type === 'backend') this.popItem(this.viewList, row)
      else {
        this.popItem(this.tableData, row)
        this.calcViewList()
        this.handleSearchChange()
      }
      row.__deleteLoading = false
      this.showMsg('success', msg, this.tipMsgs.delete.success, true)
    },
    handleDelete(index, row) {
      // 弹出确认框
      this.$confirm(
        this.tipMsgs.deleteConfirm.tip,
        this.tipMsgs.deleteConfirm.title,
        {
          confirmButtonText: this.tipMsgs.deleteConfirm.ok,
          cancelButtonText: this.tipMsgs.deleteConfirm.cancel,
          type: 'warning'
        }
      )
        .then(() => {
          // 若确认要删除
          row.__deleteLoading = true
          if (this.url && !this.onCommitDelete) {
            this.urlDelete(
              this.cleanFormData(row),
              () => {
                this.deleteSuccessCallback(row)
              },
              () => {
                row.__deleteLoading = false
                this.showMsg('error', null, this.tipMsgs.delete.fail, true)
              }
            )
          } else {
            this.onCommitDelete(
              this.cleanFormData(row),
              msg => {
                this.deleteSuccessCallback(row, msg)
              },
              msg => {
                row.__deleteLoading = false
                this.showMsg('error', msg, this.tipMsgs.delete.fail, true)
              }
            )
          }
        })
        .catch(() => {
          // row.__deleteLoading = false;
        })
    },
    handleBeginInsert() {
      if (this.insertCount > 0) {
        this.showMsg('warning', this.tipMsgs.warn.inserting)
        return
      }
      this.$refs.table.clearSort()
      const newData = {
        __edit: true,
        __insert: true,
        __deleteLoading: false,
        __editLoading: false,
        __validateMsgs: {}
      }
      for (const ii in this.tableHeaders) {
        newData[this.tableHeaders[ii].prop] = ''
      }
      this.viewList.unshift(newData)
      this.insertCount++
    },
    insertSuccessCallback(row, msg) {
      row.__edit = false
      row.__insert = false
      // 若成功且为前端分页，则插入总列表头
      if (this.type === 'frontend') {
        this.tableData.unshift(row)
        this.currentPage = 1
      }
      this.setOData(row)
      this.insertCount--
      this.total++
      this.insertLoading = false
      this.showMsg('success', msg, this.tipMsgs.insert.success)
    },
    handleCommitInsert(index, row) {
      if (!this.doValidate(row)) return
      this.insertLoading = true
      if (this.url && !this.onCommitInsert) {
        // 组件托管，自动发送post请求，新增条目
        this.urlPost(
          this.cleanFormData(row),
          resp => {
            row[this.insertIdName] = resp['data'][this.insertIdName]
            this.insertSuccessCallback(row)
          },
          () => {
            this.showMsg('error', null, this.tipMsgs.insert.fail)
          }
        )
      } else {
        this.onCommitInsert(
          this.cleanFormData(row),
          (id, msg) => {
            row[this.insertIdName] = id
            this.insertSuccessCallback(row, msg)
          },
          msg => {
            // fail
            this.showMsg('error', msg, this.tipMsgs.insert.fail)
          }
        )
      }
    },
    editSuccessCallback(row, msg) {
      row.__edit = false
      this.setOData(row)
      row.__editLoading = false
      this.showMsg('success', msg, this.tipMsgs.update.success)
    },
    handleCommitEdit(index, row) {
      if (!this.doValidate(row)) return
      row.__editLoading = true
      if (this.url && !this.onCommitEdit) {
        this.urlPut(
          this.cleanFormData(row),
          () => {
            this.editSuccessCallback(row)
          },
          () => {
            row.__editLoading = false
            this.showMsg('error', null, this.tipMsgs.update.fail)
          }
        )
      } else {
        this.onCommitEdit(
          this.cleanFormData(row),
          msg => {
            this.editSuccessCallback(row, msg)
          },
          msg => {
            // fail
            row.__editLoading = false
            this.showMsg('error', msg, this.tipMsgs.update.fail)
          }
        )
      }
    },
    handleCancelEdit(index, row) {
      if (row.__editLoading || this.insertLoading) {
        this.showMsg('warning', this.tipMsgs.warn.waiting)
        return
      }
      if (row.__insert) {
        // 取消新增
        const i = this.viewList.indexOf(row)
        if (i > -1) {
          this.viewList.splice(i, 1)
        }
        this.insertCount--
      } else {
        row.__edit = false
        // 从oData恢复数据
        this.myExtend(row, row.oData)
      }
    },
    handleSizeChange(val) {
      this.isLoading = true
      this.insertCount = 0
      this.size = val
      this.currentPage = 1
      if (this.type === 'backend') {
        // 若后端分页
        if (this.url && !this.onSizeChange) {
          this.handlePageChange(this.currentPage)
        } else {
          this.onSizeChange(
            val,
            () => {
              this.isLoading = false
            },
            msg => {
              this.showMsg('error', msg, this.tipMsgs.size.fail, true)
            }
          )
        }
      } else this.calcViewList()
    },
    handleSortChange(info) {
      this.isLoading = true
      this.currentPage = 1
      this.sortInfo = info
      if (this.type === 'backend') {
        // 后端分页的情况
        if (this.url && !this.onSortChange) {
          this.handlePageChange(this.currentPage)
        } else {
          this.onSortChange(
            { prop: info.prop, order: info.order },
            () => {
              this.isLoading = false
            },
            msg => {
              this.isLoading = false
              this.showMsg('error', msg, this.tipMsgs.sort.fail)
            }
          )
        }
      } else {
        // 前端分页的情况
        if (info.order && info.prop) {
          // 如果有排序
          this.tableData.sort((a, b) => {
            if (info.order === 'ascending') {
              return a[info.prop] > b[info.prop] ? 1 : -1
            } else if (info.order === 'descending') {
              return a[info.prop] > b[info.prop] ? -1 : 1
            }
          })
          this.calcViewList()
        } else {
          this.isLoading = false
        }
      }
    },
    handleSearchChange() {
      this.isLoading = true
      this.currentPage = 1

      if (this.type === 'backend') {
        // 后端分页时
        if (this.url && !this.onSearchChange) {
          this.handlePageChange(this.currentPage)
        } else {
          this.onSearchChange(
            this.searchKw,
            () => {
              this.isLoading = false
            },
            msg => {
              this.isLoading = false
              this.showMsg('error', msg, this.tipMsgs.search.fail)
            }
          )
        }
      } else if (this.searchKw) {
        // 前端分页时
        this.filterData = this.tableData.filter(data => {
          for (const i in this.tableHeaders) {
            const propName = this.tableHeaders[i].prop
            if (data[propName] && data[propName].indexOf(this.searchKw) > -1) {
              return true
            }
          }
        })
        this.total = this.filterData.length
        this.calcViewList()
      } else {
        this.total = this.tableData.length
        this.calcViewList()
      }
    },
    handleRowClick(row) {
      if (this.onRowClick) {
        // this.onRowClick(this.cleanFormData(row))
        this.onRowClick(row)
      }
    },
    handleSelectionChange(val) {
      if (this.onSelectionChange) {
        const newVal = []
        for (let i = 0, length = val.length; i < length; i++) {
          newVal.push(this.cleanFormData(val[i]))
        }
        this.onSelectionChange(newVal)
      }
    },
    toggleRowSelection(row, selected) {
      this.$refs.table.toggleRowSelection(row, selected)
    },
    clearSelection() {
      this.$refs.table.clearSelection()
    }
  }
}
</script>

<style>
.datatable .el-pagination__total,
.datatable .el-pagination__sizes {
  float: left;
}

.datatable .search-bar > input {
  height: 36px;
}
</style>

<style scoped>
.datatable {
  position: relative;
  /* padding: 5px; */
}

.datatable .insert-begin-button {
  padding: 10px 15px;
  margin-bottom: 5px;
}

.datatable .search-bar {
  position: absolute;
  right: 5px;
  top: 5px;
  width: 50%;
  max-width: 300px;
}

.datatable .has-search-bar {
  margin-top: 41px;
}

.el-pagination {
  margin-top: 10px;
  text-align: right;
}

.error-message {
  font-size: 12px;
  color: #f56c6c;
}

.el-pagination {
  display: inline-block;
  flex-grow: 1;
  margin-top: 0px;
}

.bottom-bar {
  display: flex;
  align-items: center;
  margin-top: 10px;
}
</style>
