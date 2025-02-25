<template>
  <div class="ds-move-tree">
    <el-input
      v-model="filterText"
      size="small"
      :placeholder="$t('commons.search')"
      prefix-icon="el-icon-search"
      style="margin-bottom: 16px"
      clearable
      class="main-area-input"
    />
    <div class="tree" v-loading="loading">
      <el-tree
        ref="datasetTreeRef"
        class="de-tree"
        :current-node-key="checkedTable ? checkedTable.id : ''"
        :default-expanded-keys="expandedArray"
        :data="data"
        node-key="id"
        highlight-current
        :expand-on-click-node="true"
        :filter-node-method="filterNode"
        @node-click="nodeClick"
      >
        <span
          v-if="data.modelInnerType === 'group'"
          slot-scope="{ data }"
          class="custom-tree-node"
        >
          <span style="display: flex; flex: 1; width: 0">
            <span v-if="data.modelInnerType === 'group'">
              <svg-icon icon-class="scene"/>
            </span>
            <span
              style="
                margin-left: 6px;
                white-space: nowrap;
                overflow: hidden;
                text-overflow: ellipsis;
              "
              :title="data.name"
              >{{ data.name }}</span
            >
          </span>
        </span>
        <span v-else slot-scope="{ data }" class="custom-tree-node-list">
          <span :id="data.id" style="display: flex; flex: 1; width: 0">
            <span>
              <svg-icon
                :icon-class="`ds-${data.modelInnerType}`"
                :class="`ds-icon-${data.modelInnerType}`"
              />
            </span>
            <span
              style="
                margin-left: 6px;
                white-space: nowrap;
                overflow: hidden;
                text-overflow: ellipsis;
              "
              :title="data.name"
              >{{ data.name }}</span
            >
          </span>
        </span>
      </el-tree>
    </div>
  </div>
</template>

<script>
import { queryAuthModel } from '@/api/authModel/authModel'
import { post } from '@/api/dataset/dataset'
export default {
  name: 'DatasetGroupSelectorTree',
  props: {
    fixHeight: {
      type: Boolean,
      required: false,
      default: false
    },
    customType: {
      type: Array,
      required: false,
      default: null
    },
    mode: {
      type: Number,
      required: false,
      default: -1
    },
    type: {
      type: String,
      required: false,
      default: null
    },
    unionData: {
      type: Array,
      required: false,
      default: null
    },
    checkedList: {
      type: Array,
      required: false,
      default: null
    },
    table: {
      type: Object,
      required: false,
      default: null
    },
    showMode: {
      type: String,
      required: false,
      default: null
    },
    privileges: {
      type: String,
      required: false,
      default: 'use'
    },
    clearEmptyDir: {
      type: Boolean,
      required: false,
      default: false
    },
    checkedTable: {
      type: Object,
      required: false,
      default: null
    }
  },
  data() {
    return {
      searchPids: [], // 查询命中的pid
      loading: false,
      filterText: '',
      searchType: 'all',
      searchMap: {
        all: this.$t('commons.all'),
        folder: this.$t('commons.folder')
      },
      sceneMode: false,
      search: '',
      data: [],
      tableData: [],
      tables: [],
      currGroup: null,
      expandedArray: [],
      groupForm: {
        name: '',
        pid: '0',
        level: 0,
        type: '',
        children: [],
        sort: 'type desc,name asc'
      },
      tableForm: {
        name: '',
        sort: 'type asc,create_time desc,name asc'
      },
      dsLoading: false,
      treeProps: {
        label: 'name',
        children: 'children',
        isLeaf: 'isLeaf',
        id: 'id',
        parentId: 'pid'
      },
      isTreeSearch: false,
      treeStyle: this.fixHeight
        ? {
            height: '300px',
            overflow: 'auto'
          }
        : {}
    }
  },
  computed: {},
  watch: {
    unionData: function () {
      this.unionDataChange()
    },
    table: function () {
      this.treeNode()
    },
    filterText(val) {
      this.searchPids = []
      this.$refs.datasetTreeRef.filter(val)
    },
    searchType(val) {
      this.searchPids = []
      this.$refs.datasetTreeRef.filter(this.filterText)
    }
  },
  mounted() {
    this.treeNode()
    this.initExpand()
  },
  created() {},
  methods: {
    close() {
      this.editGroup = false
      this.groupForm = {
        name: '',
        pid: '0',
        level: 0,
        type: '',
        children: [],
        sort: 'type desc,name asc'
      }
    },
    closeTable() {
      this.editTable = false
      this.tableForm = {
        name: ''
      }
    },
    initExpand() {
      if (this.checkedTable && this.checkedTable.pid) {
        this.expandedArray.push(this.checkedTable.pid)
      }
    },
    treeNode(cache) {
      const modelInfo = localStorage.getItem('dataset-tree')
      const userCache = modelInfo && cache
      if (userCache) {
        this.data = JSON.parse(modelInfo)
      }
      this.customType ? this.customType.push('group') : null
      this.loading = true;
      queryAuthModel(
        {
          modelType: 'dataset',
          privileges: this.privileges,
          datasetMode: this.mode,
          clearEmptyDir: this.clearEmptyDir,
          mode: this.mode < 0 ? null : this.mode,
          modelInnerTypeArray: this.customType
        },
        !userCache
      ).then((res) => {
        if (cache) {
          localStorage.setItem('dataset-tree', JSON.stringify(res.data))
        }
        if (!userCache) {
          this.data = res.data
        }
      }).finally(() => {
        this.loading = false
      })
    },
    nodeClick(data, node) {
      if (data.modelInnerType !== 'group') {
        this.sceneClick(data, node)
      }
      if (node.expanded) {
        this.expandedArray.push(data.id)
      } else {
        const index = this.expandedArray.indexOf(data.id)
        if (index > -1) {
          this.expandedArray.splice(index, 1)
        }
      }
    },
    back() {
      this.sceneMode = false
    },
    sceneClick(data, node) {
      if (data.disabled) {
        this.$message({
          type: 'warning',
          message: this.$t('dataset.invalid_dataset'),
          showClose: true
        })
        return
      }
      // check mode=1的数据集是否创建doris表
      if (data.mode === 1 && !this.showMode) {
        post(
          '/dataset/table/checkDorisTableIsExists/' + data.id,
          {},
          false
        ).then((response) => {
          if (response.data) {
            this.$nextTick(function () {
              this.$emit('getTable', data)
            })
          } else {
            this.$message({
              type: 'error',
              message: this.$t('dataset.invalid_table_check'),
              showClose: true
            })
            this.$emit('getTable', {})
          }
        })
      } else {
        this.$emit('getTable', data)
      }
    },
    unionDataChange() {
      if (!this.sceneMode) {
        return
      }
      if (!this.checkedList || this.checkedList.length === 0) {
        this.tableData.forEach((ele) => {
          const span = document.getElementById(ele.id).parentNode
          const div1 = span.parentNode
          const div2 = div1.parentNode
          span.style.removeProperty('color')
          div1.style.removeProperty('cursor')
          div2.style.removeProperty('pointer-events')
        })
        return
      }
      const tableList = this.tableData.map((ele) => {
        return ele.id
      })
      const unionList = this.unionData.map((ele) => {
        return ele.targetTableId
      })
      unionList.push(this.checkedList[0].tableId)
      const notUnionList = tableList
        .concat(unionList)
        .filter((v) => tableList.includes(v) && !unionList.includes(v))
      notUnionList.forEach((ele) => {
        const span = document.getElementById(ele).parentNode
        const div1 = span.parentNode
        const div2 = div1.parentNode
        span.style.setProperty('color', '#c0c4cc')
        div1.style.setProperty('cursor', 'not-allowed')
        div2.style.setProperty('pointer-events', 'none')
      })
    },
    nodeExpand(data) {
      if (data.id) {
        this.expandedArray.push(data.id)
      }
    },
    nodeCollapse(data) {
      if (data.id) {
        this.expandedArray.splice(this.expandedArray.indexOf(data.id), 1)
      }
    },
    filterNode(value, data) {
      if (!value) return true
      if (this.searchType === 'folder') {
        if (
          data.modelInnerType === 'group' &&
          data.label.indexOf(value) !== -1
        ) {
          this.searchPids.push(data.id)
          return true
        }
        if (this.searchPids.indexOf(data.pid) !== -1) {
          if (data.modelInnerType === 'group') {
            this.searchPids.push(data.id)
          }
          return true
        }
      } else {
        return data.label.indexOf(value) !== -1
      }
      return false
    },
    searchTypeClick(searchTypeInfo) {
      this.searchType = searchTypeInfo
    }
  }
}
</script>

<style scoped>
.el-divider--horizontal {
  margin: 12px 0;
}
.search-input {
  padding: 12px 0;
}
.tree-list ::v-deep .el-tree-node__expand-icon.is-leaf {
  display: none;
}
.custom-tree-node {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 14px;
  padding-right: 8px;
}
.custom-tree-node-list {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 14px;
  padding: 0 8px;
}
.custom-position {
  flex: 1;
  display: flex;
  align-items: center;
  font-size: 14px;
  flex-flow: row nowrap;
}
.form-item {
  margin-bottom: 0;
}
.title-css {
  height: 26px;
}
.title-text {
  line-height: 26px;
}
.scene-title {
  width: 100%;
  display: flex;
}
.scene-title-name {
  width: 100%;
  overflow: hidden;
  display: inline-block;
  white-space: nowrap;
  text-overflow: ellipsis;
}
.tree-style {
  padding: 10px;
  height: 100%;
  overflow-y: auto;
}
</style>
<style lang="scss">
.ds-move-tree {
  height: 100%;
  .tree {
    height: calc(100% - 115px);
    overflow-y: auto;
  }
}
</style>
