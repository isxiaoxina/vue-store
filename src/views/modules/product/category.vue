<template>
  <div>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false" :show-checkbox="true" node-key="catId"
      :default-expanded-keys="expanedkey" draggable :allow-drop="allowDrop" @node-drop="handleDrop">

      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>

        <span>
          <el-button type="text" size="mini" @click="() => append(data)" v-if="node.level <= 2">
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            edit
          </el-button>
          <el-button type="text" size="mini" @click="() => remove(node, data)" v-if="node.childNodes.length == 0">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="体量单位">
          <el-input v-model="category.produtcUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="optionCancel()">取 消</el-button>
        <el-button type="primary" @click="submiteData()">确 定</el-button>
      </span>
    </el-dialog>
  </div>

</template>

<script>
export default {
  props: [],
  data() {
    return {
      updateNodes:[],
      maxLevel: 0,
      title: "",
      dialogType: "",
      category: { name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: "", produtcUnit: "" },
      menus: [],
      expanedkey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      dialogVisible: false
    };
  },
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.data;
      });

    },
    append(data) {
      this.dialogType = "add"
      this.dialogVisible = true;
      this.category.name = "";
      this.category.parentCid = data.parentCid;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.catId = null;
      this.category.icon = "";
      this.category.produtcUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
    },
    edit(data) {
      this.dialogType = "edit";
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.category.name = data.category.name;
        this.category.catId = data.category.catId;
        this.category.icon = data.category.icon;
        this.category.produtcUnit = data.category.produtcUnit;
        this.category.parentCid = data.category.parentCid;
        this.category.sort = data.category.sort;
        this.category.showStatus = data.category.showStatus;
      });
      this.dialogVisible = true;

    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(` 是否删除该菜单?+[${data.name}]`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/product/category/delete2'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({ data }) => {
          this.$message({
            type: 'success',
            message: '删除成功!'
          });

          this.getMenus();
          this.expanedkey = [node.parent.data.catId]
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        });
      });

    },
    categoryAdd() {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {

        this.$message({
          type: 'success',
          message: '保存成功'
        });
        //关闭对话框
        this.dialogVisible = false;
        //完成之后默认扩展的key
        this.expanedkey = [this.category.parentCid];
        this.getMenus();
      });
    },
    categoryEdit() {
      var { catId, name, productUnit, icon } = this.category;
      var data = { catId, name, productUnit, icon }//key  value  都是一样的名字就这样写 然后
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {

        this.$message({
          type: 'success',
          message: '修改成功'
        });
        //关闭对话框
        this.dialogVisible = false;
        this.expanedkey = [this.category.parentCid];
        this.getMenus();
      });
    },
    submiteData() {
      if (dialogType === "add") {
        this.categoryAdd();
      }
      else if (dialogType === "edit") {
        this.categoryEdit();
        this.category.name = "";
        console.log(this.updateNodes);
      }
    },
    optionCancel() {
      this.dialogVisible = false;
    },
    allowDrop(draggingNode, dropNode, type) {
      //判断被拖动的当前节点以及所在的父节点总层数不能大于3
      var level = this.countNodeLevel(draggingNode.data);

      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
     

      if (type == "inner") {
        return (deep + dropNode.level) <= 3;
      } else {
        return (deep + dropNode.parent.level) <= 3;
      }
         console.log("ssssssssssssssssss");
      return false;
    },
    //求最大深度的方法   用层级来算
    countNodeLevel(node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          //递归 继续算最大深度  把一个子节点传给这个递归方法  
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, type, ev) {

      //当前节点最新的父节点id 
      let pCid = 0
      //存储拖拽进去的兄弟节点
      let siblings = null;
      //如果是转换顺序不改变层级 就把当前最新父节点的id变成最新的
      if (type == "before" || type == "after") {
        pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        //如果是inner 往里面拖 他的父节点 就是他往哪个里面拖 哪个就是他的父id  
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      //当前拖拽节点的排序
      for (let i = 0; i < siblings.length; i++) {
        //如果遍历的是当前正在拖拽的节点  除了要改顺序 还要改父id为最新的父id
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level
          //如果当前拖拽的节点的新层级跟原来的没被拖拽之前的层级不相等  就设置他的层级
          if (siblings[i].level != draggingNode.level) {
            // if (type == "before" || type == "after") {
            //       catLevel=dropNode.level  //是换位置  就设置新节点的层级是兄弟的层级
            // } else {
            //   //如果在里面 就把层级加1
            //   catLevel=dropNode.level+1
            // }

            //修改当前层级
            catLevel = siblings[i].level;
            //修改子节点的层级
            this.updateChildLevel(siblings[i])
          }
          //从上到下获得顺序
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: "菜单拖拽成功",
          type: "success"
        })
        //刷新菜单
        this.getMenus();
        //展开的id
        this.expanedkey = [pCid];
        this.updateNodes = [];
        this.maxLevel = 0;

      });

    },
    updateChildLevel(node) {
      if (node.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          //这是孩子节点的数据 塞到数组里面
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({ catId: cNode.catId, catLevel: node.childNodes[i].level })
        }
      }

    }
  },

  created() {
    this.getMenus();

  },
};
</script>
