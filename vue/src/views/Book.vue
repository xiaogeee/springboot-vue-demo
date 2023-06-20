<template>
  <div style="padding: 10px">
    <!-- 功能区域 -->
    <div style="margin: 10px 0">
      <el-button type="primary" @click="add" v-if="user.role === 1">新增</el-button>
      <el-popconfirm v-if="user.role === 1" title="确定删除吗？" @confirm="deleteBatch">
        <template #reference>
          <el-button type="danger">批量删除</el-button>
        </template>
      </el-popconfirm>
    </div>

    <!-- 搜索区域 -->
    <div style="margin: 10px 0">
      <el-input v-model="search" placeholder="请输入关键字" style="width: 20%" clearable></el-input>
      <el-button type="primary" style="margin-left: 5px" @click="load">查询</el-button>
      <el-button type="primary" style="margin-left: 5px" @click="add">发布</el-button>
      <el-button type="primary" style="margin-left: 5px" @click="deleteBatch">删除</el-button>
      <el-button type="primary" style="margin-left: 5px" @click="save">上传</el-button>
      <el-button type="primary" style="margin-left: 5px" @click="update">更新</el-button>
    </div>

    <el-table
        v-loading="loading"
        :data="tableData"
        border
        stripe
        style="width: 100%"
        @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" width="55"></el-table-column>
      <el-table-column prop="id" label="ID" sortable></el-table-column>
      <el-table-column prop="name" label="任务名称"></el-table-column>
      <el-table-column prop="price" label="工程难度"></el-table-column>
      <el-table-column prop="author" label="地点"></el-table-column>
      <el-table-column prop="createTime" label="截止时间"></el-table-column>
      <el-table-column label="现场图片">
        <template #default="scope">
          <el-image
              style="width: 100px; height: 100px"
              :src="scope.row.cover"
              :preview-src-list="[scope.row.cover]"
          ></el-image>
        </template>
      </el-table-column>
      <el-table-column label="操作" width="240">
        <template #default="scope">
          <el-button type="primary" size="mini" @click="buy(scope.row.id)">领取任务</el-button>
          <el-button size="mini" @click="handleEdit(scope.row)" v-if="user.role === 1">编辑</el-button>
          <el-popconfirm title="确定删除吗？" @confirm="handleDelete(scope.row.id)" v-if="user.role === 1">
            <template #reference>
              <el-button size="mini" type="danger">删除</el-button>
            </template>
          </el-popconfirm>
        </template>
      </el-table-column>
    </el-table>

    <div style="margin: 10px 0">
      <el-pagination
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page="currentPage"
          :page-sizes="[5, 10, 20]"
          :page-size="pageSize"
          layout="total, sizes, prev, pager, next, jumper"
          :total="total"
      ></el-pagination>

      <el-dialog title="提示" v-model="dialogVisible" width="30%">
        <el-form :model="form" label-width="120px">
          <el-form-item label="任务名称">
            <el-input v-model="form.name" style="width: 80%"></el-input>
          </el-form-item>
          <el-form-item label="工程难度">
            <el-input v-model="form.price" style="width: 80%"></el-input>
          </el-form-item>
          <el-form-item label="地点">
            <el-input v-model="form.author" style="width: 80%"></el-input>
          </el-form-item>
          <el-form-item label="截止时间">
            <el-date-picker v-model="form.createTime" value-format="YYYY-MM-DD" type="date" style="width: 80%" clearable></el-date-picker>
          </el-form-item>
          <el-form-item label="现场图片">
            <el-upload ref="upload" :action="filesUploadUrl" :on-success="filesUploadSuccess">
              <el-button type="primary">点击上传</el-button>
            </el-upload>
          </el-form-item>
        </el-form>
        <template #footer>
          <span class="dialog-footer">
            <el-button @click="dialogVisible = false">取 消</el-button>
            <el-button type="primary" @click="save">确 定</el-button>
          </span>
        </template>

      </el-dialog>
    </div>
  </div>
</template>

<script>
import request from "@/utils/request";

export default {
  name: "Book",
  components: {},
  data() {
    return {
      user: {},
      loading: true,
      form: {},
      dialogVisible: false,
      search: "",
      currentPage: 1,
      pageSize: 10,
      total: 0,
      tableData: [],
      selectedBookIds: [],
      selectedBook: {},
      filesUploadUrl: "http://" + window.server.filesUploadUrl + ":9090/files/upload",
    };
  },
  created() {
    let userStr = sessionStorage.getItem("user") || "{}";
    this.user = JSON.parse(userStr);
    // 请求服务端，确认当前登录用户的合法信息
    request.get("/user/" + this.user.id).then((res) => {
      if (res.code === "0") {
        this.user = res.data;
      }
    });

    this.load();
  },
  methods: {
    buy(bookId) {
      request.get("/order/buy/" + bookId).then((res) => {
        this.$message.success("领取成功！");
      });
    },
    deleteBatch() {
      if (this.selectedBookIds.length === 0 && Object.keys(this.selectedBook).length === 0) {
        this.$message.warning("请选择数据！");
        return;
      }
      const idsToDelete = [...this.selectedBookIds];
      if (Object.keys(this.selectedBook).length !== 0) {
        idsToDelete.push(this.selectedBook.id);
      }
      request
          .post("/book/deleteBatch", idsToDelete)
          .then((res) => {
            if (res.code === "0") {
              this.$message.success("批量删除成功");
              this.load();
            } else {
              this.$message.error(res.msg);
            }
          });
    },

    /*deleteBatch() {
      if (this.selectedBookIds.length === 0) {
        this.$message.warning("请选择数据！");
        return;
      }
      request.post("/book/deleteBatch", this.selectedBookIds).then((res) => {
        if (res.code === "0") {
          this.$message.success("批量删除成功");
          this.load();
        } else {
          this.$message.error(res.msg);
        }
      });
    },*/
   /* handleSelectionChange(val) {
      this.selectedBookIds = val.map((v) => v.id); // [{id,name}, {id,name}] => [id,id]
    },*/
    handleSelectionChange(val) {
      this.selectedBook = val[0] || {}; // 仅选择第一条数据作为选中的书籍
    },

    filesUploadSuccess(res) {
      console.log(res);
      this.form.cover = res.data;
    },
    load() {
      this.loading = true;
      request
          .get("/book", {
            params: {
              pageNum: this.currentPage,
              pageSize: this.pageSize,
              search: this.search,
            },
          })
          .then((res) => {
            this.loading = false;
            this.tableData = res.data.records;
            this.total = res.data.total;
          });
    },
    add() {
      this.dialogVisible = true;
      this.form = {};
      if (this.$refs.upload) {
        this.$refs.upload.clearFiles(); // 清除历史文件列表
      }
    },
    save() {
      if (this.form.id) {
        // 更新
        request.put("/book", this.form).then((res) => {
          console.log(res);
          if (res.code === "0") {
            this.$message({
              type: "success",
              message: "更新成功",
            });
          } else {
            this.$message({
              type: "error",
              message: res.msg,
            });
          }
          this.load(); // 刷新表格的数据
          this.dialogVisible = false; // 关闭弹窗
        });
      } else {
        // 新增
        request.post("/book", this.form).then((res) => {
          console.log(res);
          if (res.code === "0") {
            this.$message({
              type: "success",
              message: "发布成功",
            });
          } else {
            this.$message({
              type: "error",
              message: res.msg,
            });
          }

          this.load(); // 刷新表格的数据
          this.dialogVisible = false; // 关闭弹窗
        });
      }
    },
    update() {
      if (!this.selectedBook || !Object.keys(this.selectedBook).length) {
        this.$message.error("请选择需要修改的数据！");
        return;
      }
      this.dialogVisible = true; // 打开对话框
      this.form = { ...this.selectedBook }; // 将选中的书籍数据赋值给表单数据
    },



    /*update(bookId) {
      if (!bookId) {
        this.$message.error("请选择需要修改的数据！");
        return;
      }
      this.loading = true;
      request
          .put("/book/" + bookId, this.form)
          .then((res) => {
            if (res.code === "0") {
              this.$message({
                type: "success",
                message: "修改成功",
              });
              this.dialogVisible = false;
              this.load(); // 更新数据列表
            } else {
              this.$message({
                type: "error",
                message: res.msg,
              });
            }
          })
          .catch((error) => {
            console.log(error);
            this.$message.error("修改失败，请检查网络设置！");
          })
          .finally(() => {
            this.loading = false;
          });
    },*/
  /* handleEdit(row) {
      this.form = { ...row };
      this.dialogVisible = true;
      this.$nextTick(() => {
        if (this.$refs.upload) {
          this.$refs.upload.clearFiles(); // 清除历史文件列表
        }
      });
    },*/
    handleEdit(row) {
      this.selectedBook = { ...row };
      this.dialogVisible = true;
      this.$nextTick(() => {
        if (this.$refs.upload) {
          this.$refs.upload.clearFiles(); // 清除历史文件列表
        }
      });
    },


    handleDelete(id) {
      console.log(id);
      request.delete("/book/" + id).then((res) => {
        if (res.code === "0") {
          this.$message({
            type: "success",
            message: "删除成功",
          });
        } else {
          this.$message({
            type: "error",
            message: res.msg,
          });
        }
        this.load(); // 删除之后重新加载表格的数据
      });
    },
    handleSizeChange(pageSize) {
      // 改变当前每页的个数触发
      this.pageSize = pageSize;
      this.load();
    },
    handleCurrentChange(pageNum) {
      // 改变当前页码触发
      this.currentPage = pageNum;
      this.load();
    },
  },
};
</script>

<style scoped>
.el-pagination {
  margin-top: 10px;
  text-align: right;
}
</style>
