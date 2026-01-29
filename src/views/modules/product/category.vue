<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    
    <!-- 批量保存有bug 暂时不支持 -->
    <!-- <el-button v-if="draggable" @click="butchSave">批量保存</el-button> -->

    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>

          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>

          <el-button
            v-if="node.childNodes.length <= 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: "", // 添加 | 修改
      dialogType: "", // add | edit
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
        catId: null,
      },
      dialogVisible: false,
      menus: [],
      expandKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  // 计算属性类似于data 概念
  computed: {},
  // 监控data 中的数据变化
  watch: {},

  // 方法集合
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        // console.log("获取菜单列表", data.data);
        this.menus = data.data;
      });
    },

    batchDelete() {
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      if (checkedNodes.length === 0) {
        this.$message.warning("请先选择要删除的分类");
        return;
      }
      let catIds = checkedNodes.map((item) => item.catId);
      this.$confirm(`是否删除选中的分类？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "分类删除成功",
                type: "success",
              });
              // 刷新菜单
              this.getMenus();
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {});
    },

    // batchSave() { 
    //   // 批量保存有问题 要去重(或者说要有先后 对每个节点只保存最后一次的数据) 先不写了
    // },

    // 拖拽完成监听事件
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // console.log("拖拽完成", draggingNode, dropNode, dropType);
      // 1.当前节点最新父id
      let pCid = 0;
      let siblings = null;
      this.updateNodes = [];

      if (dropType === "inner") {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      }
      // 2.当前节点最新顺序
      // 3.当前节点最新层级 catLevel是数据库的数据 level是是前端框架的数据（本身就会变）
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历到当前节点 还要改父id
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            // 层级改变了 直接把level同步给catLevel 包括所有子节点
            catLevel = siblings[i].level;
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }

      // console.log("updateNodes", this.updateNodes);
      this.$http({
        url: this.$http.adornUrl("/product/category/update/batch"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "分类排序更新成功",
            type: "success",
          });
          // 刷新菜单
          this.getMenus();
          this.expandKey = [pCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    updateChildNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },

    allowDrop(draggingNode, dropNode, type) {
      // 初始化
      this.maxLevel = draggingNode.level;
      // console.log("allowDrop", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode);
      // 被拖节点下挂层数
      let deep = this.maxLevel - draggingNode.level + 1;
      // console.log("deep", deep);
      if (type === "inner") {
        // console.log("dropNode.level", dropNode.level);
        return deep + dropNode.level <= 3;
      } else {
        // console.log("dropNode.parent.level", dropNode.parent.level);
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      // 找到所有子节点 求最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },

    edit(data) {
      console.log("要修改的数据", data);
      this.dialogType = "edit";
      this.title = "修改分类";
      this.dialogVisible = true;

      // 发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        // 请求成功
        console.log("要回显的数据", data);
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.catId = data.data.catId;
        this.category.parentCid = data.data.parentCid;
        this.category.catLevel = data.data.catLevel;
        this.category.showStatus = data.data.showStatus;
        this.category.sort = data.data.sort;
      });
    },
    append(data) {
      console.log("append", data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      // 清数据 防止修改之后又新增 回显数据不对
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.showStatus = 1;
      this.category.sort = 0;
    },

    submitData() {
      if (this.dialogType === "add") {
        this.addCategory();
      } else if (this.dialogType === "edit") {
        this.editCategory();
      }
    },

    // 修改三级分类
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      var data = {
        catId: catId,
        name: name,
        icon: icon,
        productUnit: productUnit,
      };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "菜单修改成功",
            type: "success",
          });
          // 关闭对话框
          this.dialogVisible = false;
          // 刷新菜单
          this.getMenus();
          // 默认展开
          this.expandKey = [this.category.parentCid];
        }
      });
    },

    // 添加三级分类
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "菜单保存成功",
            type: "success",
          });
          // 关闭对话框
          this.dialogVisible = false;
          // 刷新菜单
          this.getMenus();
          // 默认展开
          this.expandKey = [this.category.parentCid];
        }
      });
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "菜单删除成功",
                type: "success",
              });
              // 刷新菜单
              this.getMenus();
              // 默认展开
              this.expandKey = [node.parent.data.catId];
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {});

      console.log("remove", node, data);
    },
  },

  // 生命周期- 创建完成（可以访问当前this 实例）
  created() {
    this.getMenus();
  },
  // 生命周期- 挂载完成（可以访问DOM 元素）
  mounted() {},
  beforeCreate() {}, // 生命周期- 创建之前
  beforeMount() {}, // 生命周期- 挂载之前
  beforeUpdate() {}, // 生命周期- 更新之前
  updated() {}, // 生命周期- 更新之后
  beforeDestroy() {}, // 生命周期- 销毁之前
  destroyed() {}, // 生命周期- 销毁完成
  activated() {}, // 如果页面有keep-alive 缓存功能，这个函数会触发
};
</script>
  <style scoped>
</style>