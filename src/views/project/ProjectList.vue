<template>
  <a-row :gutter="10">
    <a-col :md="8" :sm="24">
      <a-card :bordered="false">
        <!-- 按钮操作区域 -->
        <a-row style="margin-left: 14px">
          <a-button @click="handleAdd(2)" type="primary">添加子项目</a-button>
          <a-button @click="handleAdd(1)" type="default">添加父项目</a-button>
          <a-button title="删除多条数据" @click="batchDel" type="default">批量删除</a-button>
          <a-button @click="refresh" type="default" icon="reload" :loading="loading">刷新</a-button>
        </a-row>
        <div style="background: #fff;padding-left:16px;height: 100%; margin-top: 5px">
          <a-alert type="info" :showIcon="true">
            <div slot="message">
              当前选择：
              <a v-if="this.currSelected.title">{{ getCurrSelectedTitle() }}</a>
              <a
                v-if="this.currSelected.title"
                style="margin-left: 10px"
                @click="onClearSelected"
              >取消选择</a>
            </div>
          </a-alert>
          <!-- <a-input-search
            @search="onSearch"
            style="width:100%;margin-top: 10px"
            placeholder="请输入项目名称"
          />-->
          <!-- 树-->
          <a-col :md="10" :sm="24">
            <template>
              <a-dropdown :trigger="[this.dropTrigger]" @visibleChange="dropStatus">
                <span style="user-select: none">
                  <a-tree
                    checkable
                    multiple
                    @select="onSelect"
                    @check="onCheck"
                    @rightClick="rightHandle"
                    :selectedKeys="selectedKeys"
                    :checkedKeys="checkedKeys"
                    :treeData="projectTree"
                    :checkStrictly="true"
                    :expandedKeys="iExpandedKeys"
                    :autoExpandParent="autoExpandParent"
                    @expand="onExpand"
                  />
                </span>
                <!--新增右键点击事件,和增加添加和删除功能-->
                <a-menu slot="overlay">
                  <a-menu-item @click="handleAdd(3)" key="1">添加</a-menu-item>
                  <a-menu-item @click="handleDelete" key="2">删除</a-menu-item>
                </a-menu>
              </a-dropdown>
            </template>
          </a-col>
        </div>
      </a-card>
    </a-col>
    <a-col :md="16" :sm="24">
      <a-card :bordered="false">
        <a-tabs defaultActiveKey="1" @change="callback">
          <a-tab-pane tab="基本信息" key="1" forceRender>
            <a-form :form="form">
              <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="项目名称">
                <a-input v-decorator="['projectName', {'initialValue':''}]" />
              </a-form-item>
              <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="排序">
                <a-input v-decorator="['sort', {'initialValue':''}]" />
              </a-form-item>
              <!-- <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="项目介绍">
                <a-textarea v-decorator="['projectDetail', {'initialValue':''}]" />
              </a-form-item>-->
              <a-form-item :labelCol="labelCol" :wrapperCol="wrapperCol" label="项目介绍">
                <editor
                  v-model="projectDetail"
                  :init="init"
                  @onClick="onClick"
                ></editor>
              </a-form-item>
            </a-form>
            <div class="anty-form-btn">
              <a-button @click="submitCurrForm" type="primary" htmlType="button" icon="form">修改并保存</a-button>
            </div>
          </a-tab-pane>
          <a-tab-pane tab="项目进度" key="2" forceRender>
            <JdList ref="JdList"></JdList>
          </a-tab-pane>
          <a-tab-pane tab="项目财务" key="3" forceRender>
            <CwList ref="CwList"></CwList>
          </a-tab-pane>
          <a-tab-pane tab="项目收支平衡" key="4" forceRender>
            <SzphList ref="SzphList"></SzphList>
          </a-tab-pane>
        </a-tabs>
      </a-card>
    </a-col>
    <project-modal ref="projectModal" @ok="loadTree"></project-modal>
  </a-row>
</template>

<script>
import JdList from './JdList'
import CwList from './CwList'
import SzphList from './SzphList'
import ProjectModal from './modules/ProjectModal'
import { JeecgListMixin } from '@/mixins/JeecgListMixin'
import { httpAction, deleteAction, getAction } from '@/api/manage'
import pick from 'lodash.pick'
import tinymce from 'tinymce/tinymce'
import Editor from '@tinymce/tinymce-vue'
import 'tinymce/themes/silver/theme'
import 'tinymce/plugins/image'
import 'tinymce/plugins/media'
import 'tinymce/plugins/table'
import 'tinymce/plugins/lists'
import 'tinymce/plugins/contextmenu'
import 'tinymce/plugins/wordcount'
import 'tinymce/plugins/colorpicker'
import 'tinymce/plugins/textcolor'

const columns = [
  {
    title: '#',
    dataIndex: '',
    key: 'rowIndex',
    width: 60,
    align: 'center',
    customRender: function(t, r, index) {
      return parseInt(index) + 1
    }
  },
  {
    title: '项目名称',
    align: 'center',
    dataIndex: 'projectName'
  },
  {
    title: '操作',
    dataIndex: 'action',
    align: 'center',
    scopedSlots: { customRender: 'action' }
  }
]
export default {
  name: 'ProjectList',
  mixins: [JeecgListMixin],
  components: {
    JdList,
    CwList,
    SzphList,
    ProjectModal,
    Editor
  },
  data() {
    return {
      description: '项目管理管理页面',
      projectId: '',
      iExpandedKeys: [],
      projectDetail: '',
      loading: false,
      autoExpandParent: true,
      currFlowId: '',
      currFlowName: '',
      disable: true,
      treeData: [],
      visible: false,
      projectTree: [],
      rightClickSelectedKey: '',
      hiding: true,
      model: {},
      dropTrigger: '',
      depart: {},
      columns: columns,
      disableSubmit: false,
      checkedKeys: [],
      selectedKeys: [],
      autoIncr: 1,
      currSelected: {},
      form: this.$form.createForm(this),
      labelCol: {
        xs: { span: 24 },
        sm: { span: 5 }
      },
      wrapperCol: {
        xs: { span: 24 },
        sm: { span: 16 }
      },
      graphDatasource: {
        nodes: [],
        edges: []
      },
      validatorRules: {
        projectName: { rules: [{ required: true, message: '请输入项目名称' }] },
        projectDetail: { rules: [{ required: true, message: '请输入项目介绍!' }] }
      },
      url: {
        list: '/project/project/list',
        edit: 'project/project/edit',
        delete: '/project/project/delete',
        deleteBatch: '/project/project/deleteBatch',
        exportXlsUrl: 'project/project/exportXls',
        importExcelUrl: 'project/project/importExcel',
        projectTreeList: 'project/project/queryTreeList'
      },
       //初始化配置
      init: {
        language_url: '/tinymce/langs/zh_CN.js',
        language: 'zh_CN',
        skin_url: '/tinymce/skins/lightgray',
        height: 800,
        plugins: this.plugins,
        toolbar: this.toolbar,
        branding: false,
        menubar: false,
        images_upload_handler: (blobInfo, success) => {
          const img = 'data:image/jpeg;base64,' + blobInfo.base64()
          success(img)
        }
      },
    }
  },
  mounted() {
    tinymce.init({})
  },
  computed: {
    importExcelUrl: function() {
      return `${window._CONFIG['domianURL']}/${this.url.importExcelUrl}`
    }
  },
  created() {
    this.currFlowId = this.$route.params.id
    this.currFlowName = this.$route.params.name
    this.loadTree()
  },
  methods: {
    onClick(e) {
      this.$emit('onClick', e, tinymce)
    },
    callback(key) {
      console.log(key)
    },
    loadTree() {
      var that = this
      that.treeData = []
      that.projectTree = []
      let params = {}
      getAction(this.url.projectTreeList, params).then(res => {
        if (res.success) {
          for (let i = 0; i < res.result.length; i++) {
            let temp = res.result[i]
            that.treeData.push(temp)
            that.projectTree.push(temp)
            that.setThisExpandedKeys(temp)
            console.log(temp.id)
          }
          this.loading = false
        }
      })
    },
    setThisExpandedKeys(node) {
      if (node.children && node.children.length > 0) {
        this.iExpandedKeys.push(node.key)
        for (let a = 0; a < node.children.length; a++) {
          this.setThisExpandedKeys(node.children[a])
        }
      }
    },
    refresh() {
      this.loading = true
      this.loadTree()
    },
    // 右键操作方法
    rightHandle(node) {
      this.dropTrigger = 'contextmenu'
      console.log(node.node.eventKey)
      this.rightClickSelectedKey = node.node.eventKey
    },
    onExpand(expandedKeys) {
      console.log('onExpand', expandedKeys)
      // if not set autoExpandParent to false, if children expanded, parent can not collapse.
      // or, you can remove all expanded children keys.
      this.iExpandedKeys = expandedKeys
      this.autoExpandParent = false
    },
    backFlowList() {
      this.$router.back(-1)
    },
    // 右键点击下拉框改变事件
    dropStatus(visible) {
      if (visible == false) {
        this.dropTrigger = ''
      }
    },
    // 右键店家下拉关闭下拉框
    closeDrop() {
      this.dropTrigger = ''
    },
    addRootNode() {
      this.$refs.nodeModal.add(this.currFlowId, '')
    },
    batchDel: function() {
      console.log(this.checkedKeys)
      if (this.checkedKeys.length <= 0) {
        this.$message.warning('请选择一条记录！')
      } else {
        var ids = ''
        for (var a = 0; a < this.checkedKeys.length; a++) {
          ids += this.checkedKeys[a] + ','
        }
        var that = this
        this.$confirm({
          title: '确认删除',
          content: '确定要删除所选中的 ' + this.checkedKeys.length + ' 条数据?',
          onOk: function() {
            deleteAction(that.url.deleteBatch, { ids: ids }).then(res => {
              if (res.success) {
                that.$message.success(res.message)
                that.loadTree()
                that.onClearSelected()
              } else {
                that.$message.warning(res.message)
              }
            })
          }
        })
      }
    },
    /*  onSearch(value) {
      let that = this
      if (value) {
        searchByKeywords({ keyWord: value }).then(res => {
          if (res.success) {
            that.departTree = []
            for (let i = 0; i < res.result.length; i++) {
              let temp = res.result[i]
              that.departTree.push(temp)
            }
          } else {
            that.$message.warning(res.message)
          }
        })
      } else {
        that.loadTree()
      }
    }, */
    nodeModalOk() {
      this.loadTree()
    },
    nodeModalClose() {},
    hide() {
      console.log(111)
      this.visible = false
    },
    onCheck(checkedKeys, info) {
      console.log('onCheck', checkedKeys, info)
      this.hiding = false
      this.checkedKeys = checkedKeys.checked
    },
    onSelect(selectedKeys, e) {
      this.hiding = false
      let record = e.node.dataRef
      this.currSelected = Object.assign({}, record)
      this.model = this.currSelected
      this.selectedKeys = [record.key]
      this.model.pid = record.pid
      this.setValuesToForm(record)

      this.$refs.JdList.open(record)
      this.$refs.CwList.open(record)
      this.$refs.SzphList.open(record)
    },
    // 触发onSelect事件时,为部门树右侧的form表单赋值
    setValuesToForm(record) {
      if (JSON.stringify(record) == '{}') {
        this.projectDetail = ''
      } else {
        this.projectDetail = record.projectDetail
      }
      record.projectName = record.title
      //this.projectName = record.title
      this.form.setFieldsValue(pick(record, 'projectName', 'projectDetail', 'sort'))
    },
    getCurrSelectedTitle() {
      return !this.currSelected.title ? '' : this.currSelected.title
    },
    onClearSelected() {
      this.hiding = true
      this.checkedKeys = {}
      this.currSelected = {}
      this.form.resetFields()
      this.selectedKeys = []
    },
    handleAdd(num) {
      if (num == 1) {
        this.$refs.projectModal.add()
        this.$refs.projectModal.title = '新增'
      } else if (num == 2) {
        let key = this.currSelected.key
        if (!key) {
          this.$message.warning('请先选中一条记录!')
          return false
        }
        this.$refs.projectModal.add(this.selectedKeys)
        this.$refs.projectModal.title = '新增'
      }
    },
    handleDelete() {
      deleteAction(this.url.delete, { id: this.rightClickSelectedKey }).then(res => {
        if (res.success) {
          this.$message.success('删除成功!')
          this.loadTree()
        } else {
          this.$message.warning('删除失败!')
        }
      })
    },
    submitCurrForm() {
      this.form.validateFields((err, values) => {
        console.log(values)
        if (!err) {
          if (!this.currSelected.id) {
            this.$message.warning('请点击选择要修改的项目!')
            return
          }

          this.currSelected.projectDetail = this.projectDetail
          let formData = Object.assign(this.currSelected, values)
          console.log('Received values of form: ', formData)
          httpAction(this.url.edit, formData, 'put').then(res => {
            if (res.success) {
              this.$message.success('保存成功!')
              this.loadTree()
            }
          })
        }
      })
    }
  }
}
</script>
<style lang="less" scoped>
/** Button按钮间距 */
.ant-btn {
  margin-left: 3px;
}
.ant-card-body .table-operator {
  margin-bottom: 18px;
}
.ant-table-tbody .ant-table-row td {
  padding-top: 15px;
  padding-bottom: 15px;
}
.anty-row-operator button {
  margin: 0 5px;
}
.ant-btn-danger {
  background-color: #ffffff;
}

.ant-modal-cust-warp {
  height: 100%;
}
.ant-modal-cust-warp .ant-modal-body {
  height: calc(100% - 110px) !important;
  overflow-y: auto;
}
.ant-modal-cust-warp .ant-modal-content {
  height: 90% !important;
  overflow-y: hidden;
}
</style>