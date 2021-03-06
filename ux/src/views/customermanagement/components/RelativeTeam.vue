<template>
  <div class="rc-cont">
    <flexbox class="rc-head"
             direction="row-reverse">
      <el-button class="rc-head-item"
                 @click.native="handleClick('remove')"
                 type="primary">移除</el-button>
      <el-button class="rc-head-item"
                 @click.native="handleClick('edit')"
                 type="primary">编辑</el-button>
      <el-button class="rc-head-item"
                 @click.native="handleClick('add')"
                 type="primary">添加团队成员</el-button>
    </flexbox>
    <el-table :data="list"
              :height="tableHeight"
              stripe
              style="width: 100%;border: 1px solid #E6E6E6;"
              :header-cell-style="headerRowStyle"
              :cell-style="cellStyle"
              @row-click="handleRowClick"
              @selection-change="handleSelectionChange">
      <el-table-column show-overflow-tooltip
                       type="selection"
                       align="center"
                       width="55"
                       :selectable="handleSelectable">
      </el-table-column>
      <el-table-column v-for="(item, index) in fieldList"
                       :key="index"
                       show-overflow-tooltip
                       :prop="item.prop"
                       :label="item.label">
      </el-table-column>
    </el-table>
    <teams-handle :crmType="crmType"
                  title="添加团队成员"
                  :selectionList="[detail]"
                  @handle="handleCallBack"
                  :dialogVisible.sync="teamsDialogShow"></teams-handle>

    <el-dialog title="编辑权限"
               v-loading="loading"
               :visible.sync="editPermissionShow"
               :append-to-body="true"
               width="400px">
      <div class="handle-box">
        <flexbox class="handle-item">
          <div class="handle-item-name">权限：</div>
          <el-radio-group v-model="handleType">
            <el-radio :label="1">只读</el-radio>
            <el-radio :label="2">读写</el-radio>
          </el-radio-group>
        </flexbox>
      </div>
      <span slot="footer"
            class="dialog-footer">
        <el-button @click.native="editPermissionShow=false">取消</el-button>
        <el-button type="primary"
                   @click.native="handleEditConfirm">保存</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script type="text/javascript">
import { crmSettingTeam } from '@/api/customermanagement/common'
import loading from '../mixins/loading'
import TeamsHandle from './selectionHandle/TeamsHandle' // 操作团队成员
import { crmSettingTeamSave } from '@/api/customermanagement/common'

export default {
  name: 'relative-team', //相关团队  可能再很多地方展示 放到客户管理目录下
  components: {
    TeamsHandle
  },
  computed: {},
  mixins: [loading],
  data() {
    return {
      list: [],
      fieldList: [],
      tableHeight: '400px',
      teamsDialogShow: false, //是否展示添加团队成员窗口
      handleType: 1, // 权限类型
      editPermissionShow: false, //编辑权限接口展示
      selectionList: [] //勾选的数据
    }
  },
  watch: {
    id: function(val) {
      this.list = []
      this.getDetail()
    }
  },
  props: {
    /** 模块ID */
    id: [String, Number],
    /** 联系人人下 新建商机 需要联系人里的客户信息  合同下需要客户和商机信息 */
    detail: {
      type: Object,
      default: () => {
        return {}
      }
    },
    /** 没有值就是全部类型 有值就是当个类型 */
    crmType: {
      type: String,
      default: ''
    }
  },
  mounted() {
    this.fieldList.push({ prop: 'realname', width: '200', label: '姓名' })
    this.fieldList.push({ prop: 'structure_name', width: '200', label: '职位' })
    this.fieldList.push({ prop: 'group_name', width: '200', label: '团队角色' })
    this.fieldList.push({ prop: 'authority', width: '200', label: '权限' })

    this.getDetail()
  },
  activated: function() {},
  deactivated: function() {},
  methods: {
    getDetail() {
      this.loading = true
      crmSettingTeam({
        types: 'crm_' + this.crmType,
        types_id: this.id
      })
        .then(res => {
          this.loading = false
          this.list = res.data
        })
        .catch(() => {
          this.loading = false
        })
    },
    //当选择项发生变化时会触发该事件
    handleSelectionChange(val) {
      this.selectionList = val
    },
    handleClick(type) {
      if (type == 'add') {
        this.teamsDialogShow = true
      } else {
        if (this.selectionList.length == 0) {
          this.$message.error('请勾选需要操作的团队成员')
        } else {
          if (type == 'edit') {
            this.editPermissionShow = true
          } else if (type == 'remove') {
            this.$confirm('此操作将移除这些团队成员是否继续?', '提示', {
              confirmButtonText: '确定',
              cancelButtonText: '取消',
              type: 'warning'
            })
              .then(() => {
                var user_ids = this.selectionList.map(function(
                  item,
                  index,
                  array
                ) {
                  return item.id
                })
                var params = {
                  types: 'crm_' + this.crmType,
                  types_id: [this.detail[this.crmType + '_id']],
                  user_id: user_ids
                }
                params.is_del = 1

                this.loading = true
                crmSettingTeamSave(params)
                  .then(res => {
                    this.$message({
                      type: 'success',
                      message: res.data
                    })
                    this.loading = false
                    this.getDetail()
                  })
                  .catch(() => {
                    this.loading = false
                  })
              })
              .catch(() => {
                this.$message({
                  type: 'info',
                  message: '已取消删除'
                })
              })
          }
        }
      }
    },
    /** 添加操作 */
    handleCallBack(data) {
      this.getDetail()
    },
    /** 编辑操作确定 */
    handleEditConfirm() {
      var user_ids = this.selectionList.map(function(item, index, array) {
        return item.id
      })
      this.loading = true
      crmSettingTeamSave({
        types: 'crm_' + this.crmType,
        types_id: [this.detail[this.crmType + '_id']],
        user_id: user_ids,
        type: this.handleType
      })
        .then(res => {
          this.editPermissionShow = false
          this.$message({
            type: 'success',
            message: res.data
          })
          this.loading = false
          this.getDetail()
        })
        .catch(() => {
          this.loading = false
        })
    },
    //返回值用来决定这一行的 CheckBox 是否可以勾选
    handleSelectable(row, index) {
      /** row 是数据源 正式调试以他为主  index 是行 */
      if (row.type == 0) {
        // 0负责人，1只读，2读写
        return false
      }
      return true
    },
    //当某一行被点击时会触发该事件
    handleRowClick(row, column, event) {},
    /** 通过回调控制表头style */
    headerRowStyle({ row, column, rowIndex, columnIndex }) {
      return { textAlign: 'center' }
    },
    /** 通过回调控制style */
    cellStyle({ row, column, rowIndex, columnIndex }) {
      return { textAlign: 'center' }
    }
  }
}
</script>
<style lang="scss" scoped>
@import '../styles/relativecrm.scss';
</style>
