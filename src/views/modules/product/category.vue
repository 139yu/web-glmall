<template>
  <div class="app-container">
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      draggable
      :data="menuList"
      show-checkbox
      :props="defaultProps"
      :default-expand-all="false"
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
      :allow-drop="allowDrop"
      :allow-drag="allowDrag"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">添加</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">修改</el-button>
          <el-button
            v-if="node.childNodes.length ==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >删除</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="addMenuDialog" width="20%">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
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
export default {
  data() {
    return {
      updateNodes: [],
      maxLevel: 0,
      pCid: [],
      expandedKey: [],
      dialogType: "",
      title: "",
      addMenuDialog: false,
      menuList: [],
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
        catId: null
      },
      defaultProps: {
        children: "children",
        label: "name"
      }
    };
  },
  methods: {
    submitData() {
      if (this.dialogType == "save") {
        this.addMenu();
      } else {
        this.updateMenu();
      }
    },
    getMenuList() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get"
      }).then(res => {
        this.menuList = res.data.data;
        console.log(this.menuList)
      });
    },
    addMenu() {
      this.updateNodes.push(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        if (data.msg) {
          this.$message({
            message: "操作成功",
            type: "success"
          });
          this.addMenuDialog = false;
          this.getMenuList();
          this.updateNodes = [];
          this.expandedKey = [this.category.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    updateMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        if (data.msg) {
          this.$message({
            message: "操作成功",
            type: "success"
          });
          this.addMenuDialog = false;
          this.getMenuList();
          this.updateNodes = [];
          this.expandedKey = [this.category.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    edit(data) {
      this.dialogType = "update";
      this.$http({
        url: this.$http.adornUrl(`product/category/info/${data.catId}`),
        method: "get",
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.category = data.category;
        this.addMenuDialog = true;
      });
    },
    append(data) {
      this.addMenuDialog = true;
      this.dialogType = "save";
      this.title = "添加菜单";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.showStatus = 1;
      this.category.sort = 0;
      this.category.catId = null;
      this.category.name = "";
      this.category.productUnit = "";
      this.category.icon = "";
    },
    remove(node, data) {
      let ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "delete",
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              message: "菜单删除成功",
              type: "success"
            });
            this.getMenuList();
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    allowDrop(draggingNode, dropNode, type) {
      this.currentNodeLevel(draggingNode.data);
      var deep = this.maxLevel - draggingNode.data.catLevel + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    //是否允许拖动
    allowDrag(draggingNode) {
      return true;
    },
    //找到所有子节点，获取最大深度
    currentNodeLevel(data) {
      if (data.children != null && data.children.length > 0) {
        for (let i = 0; i < data.children.length; i++) {
          if (data.children[i].catLevel > this.maxLevel) {
            this.maxLevel = data.children[i].catLevel;
          }
          this.currentNodeLevel(data.children[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      //draggingNode被拖拽的结点
      //拖拽成功后需要修改被拖拽结点的层级，子节点层级，父节点id，兄弟结点排序
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid = dropNode.data.parentCid;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId == undefined ? 0 : dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let curLevel = siblings[i].level;
          //如果当前节点的层级修改了，那它的字节点层级也需要修改
          if (draggingNode.data.catLevel != curLevel) {
            //当前节点需要修改的信息：父菜单，排序，层级
            this.updateNodes.push({
              catId: siblings[i].data.catId,
              parentCid: pCid,
              sort: i,
              catLevel: curLevel
            });
            //修改子节点层级
            this.updateChildNodeLevel(siblings[i]);
          }else {
            this.updateNodes.push({
              catId: siblings[i].data.catId,
              sort: i,
            });
          }
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log(this.updateNodes);
      this.$http({
        url: this.$http.adornUrl(`/product/category/update`),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        console.log(data);
        if (data.msg) {
          this.$message({
            message: "操作成功",
            type: "success"
          });
          this.updateNodes = [];
          this.getMenuList();
        }
      });
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length != 0) {
        let data = node.childNodes;
        for (let i = 0; i < data.length; i++) {
          this.updateNodes.push({
            catId: data[i].data.catId,
            catLevel: data[i].level
          });
          if (data[i].childNodes.length != 0) {
            this.updateChildNodeLevel(data[i].childNodes);
          }
        }
      }
    },
    batchDelete(){
      let checkNodes = this.$refs.menuTree.getCheckedNodes()
      let catIds = []
      for(let i = 0; i < checkNodes.length; i++){
          catIds.push(checkNodes[i].catId)
      }
      this.$confirm(`是否批量删除【${catIds}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "delete",
            data: this.$http.adornData(catIds, false)
          }).then(({ data }) => {
            this.$message({
              message: "菜单删除成功",
              type: "success"
            });
            this.getMenuList();
          });
        })
        .catch(() => {});

    }
  },
  created() {
    this.getMenuList();
  }
};
</script>

<style>
</style>