<template>
  <div class="app-container">
    <div style="line-height: 40px; font-weight: 700; font-size: 22px; color: #a85a11; margin-left: 10px;"> 课程列表</div>
    <!--查询表单-->
    <el-form :inline="true" class="demo-form-inline">
      <el-form-item>
        <el-input v-model="courseQuery.title" placeholder="课程名称"/>
      </el-form-item>
      <el-form-item>
        <el-select v-model="courseQuery.psychologistId" placeholder="讲师名称" clearable @change="getList()" style="width: 160px">
          <el-option v-for="psychologist in psychologistList" :key="psychologist.id" :label="psychologist.name" :value="psychologist.id" />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-select v-model="courseQuery.subjectParentId" placeholder="课程一级分类" @change="subjectLevelOneChanged" style="width: 160px">
          <el-option v-for="subject in subjectOneList" :key="subject.id" :label="subject.title" :value="subject.id"/>
        </el-select>

        <el-select v-model="courseQuery.subjectId" placeholder="课程二级分类" clearable @change="getList()" style="width: 160px">
          <el-option v-for="subject in subjectTwoList" :key="subject.id" :label="subject.title" :value="subject.id" />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-select v-model="courseQuery.status" clearable placeholder="课程状态" @change="getList()" style="width: 120px">
          <el-option value="Normal" label="已发布"/>
          <el-option value="Draft" label="未发布"/>
        </el-select>
      </el-form-item>

      <el-button type="primary" plain="true" icon="el-icon-search" @click="getList()">查询</el-button>
      <el-button type="default" plain="true" @click="resetData()">清空</el-button>
      <el-button v-waves :loading="downloadLoading" class="filter-item" type="primary" icon="el-icon-download"
                 @click="handleDownload">
        导出
      </el-button>
    </el-form>

    <!-- 表格 -->
    <el-table
      :data="list"
      border
      fit
      highlight-current-row>

      <el-table-column
        label="序号"
        width="100"
        align="center">
        <template slot-scope="scope">
          {{ (page - 1) * limit + scope.$index + 1 }}
        </template>
      </el-table-column>

      <el-table-column label="封面" width="250" align="center">
        <template slot-scope="scope">
          <img :src="scope.row.cover" alt="scope.row.title" width="100%">
        </template>
      </el-table-column>

      <el-table-column label="课程信息">
        <template slot-scope="scope">
          <a href="">{{ scope.row.title }}</a>
          <p>
            课时：{{ scope.row.lessonNum }}
          </p>
          <p>
            浏览：{{ scope.row.viewCount }} /
            销售数量：{{ scope.row.buyCount }}
          </p>
        </template>
      </el-table-column>

      <el-table-column label="价格(元)" width="130" align="center" >
        <template slot-scope="scope">
          <!-- {{ typeof '0' }}  {{ typeof 0 }} {{ '0' == 0 }} -->
          <!-- {{ typeof scope.row.price }}
          {{ typeof Number(scope.row.price) }}
          {{ typeof Number(scope.row.price).toFixed(2) }} -->

          <el-tag v-if="Number(scope.row.price) === 0" type="success">免费</el-tag>

          <!-- 前端解决保留两位小数的问题 -->
          <!-- <el-tag v-else>{{ Number(scope.row.price).toFixed(2) }}</el-tag> -->

          <!-- 后端解决保留两位小数的问题，前端不用处理 -->
          <el-tag v-else>{{ scope.row.price }}</el-tag>
        </template>
      </el-table-column>

      <el-table-column prop="status" label="课程状态" width="130" align="center" >
        <template slot-scope="scope">
          <el-tag :type="scope.row.status === 'Draft' ? 'warning' : 'success'">{{ scope.row.status === 'Draft' ? '未发布' : '已发布' }}</el-tag>
        </template>
      </el-table-column>

      <el-table-column prop="gmtCreate" label="添加时间" width="170" />

      <el-table-column label="操作" align="center" style="height: 100%;">
        <template slot-scope="scope">
          <router-link :to="'/course/info/'+scope.row.id">
            <el-button type="primary" plain="true" size="mini" icon="el-icon-edit">编辑课程基本信息</el-button>
          </router-link>
          <router-link :to="'/course/chapter/'+scope.row.id">
            <el-button type="primary" plain="true" size="mini" icon="el-icon-edit">编辑课程大纲信息</el-button>
          </router-link>
          <el-button type="danger" plain="true" size="mini" icon="el-icon-delete" @click="removeDataById(scope.row.id)">删除课程信息</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页 -->
    <el-pagination :current-page="page" :page-size="limit" :total="total" style="padding: 30px 0; text-align: center;"
                   layout="total, prev, pager, next, jumper" @current-change="getList" />
  </div>
</template>
<script>
  //引入调用course.js文件
  import courseApi from '@/api/Edu/course/course.js'

  //引入调用psychologist.js文件
  import psychologistApi from '@/api/Edu/psychologist/psychologist.js'

  //引入调用subject.js文件
  import subjectApi from '@/api/Edu/subject/subject.js'

  export default {
    //写核心代码的位置
    data() { //定义变量和初始值
      return {
        downloadLoading: false,
        list: null, //查询之后接口返回集合
        page: 1, //当前页
        limit: 10, //每页记录数
        total: 0, //总记录数
        courseQuery: {} ,//条件封装的对象 v-model双向绑定
        subjectOneList: [], //数组封装所有一级分类信息
        subjectTwoList: [], //数组封装所有二级分类信息
        psychologistList: [] //讲师列表
      }
    },
    created() { //页面渲染之前执行，调用method定义的方法
      //调用
      this.getList();
      //课程讲师初始化
      this.initPsychologistList();
      //初始化分类列表
      this.initSubjectList()
    },
    methods: { //创建具体的方法，调用course.js定义的方法
      getList(page = 1) { //讲师列表的方法
        this.page = page;
        courseApi.findAll(this.page, this.limit, this.courseQuery)
          .then(response => { //请求成功
            //response接口返回的数据
            this.list = response.data.records;
            this.total = response.data.total
          })
          .catch(error => { //请求失败
            console.log(error)
          })
      },

      // 获取讲师列表
      initPsychologistList() {
        psychologistApi.getAllList().then(response => {
          this.psychologistList = response.data.items
        })
      },
      //初始化课程分类
      initSubjectList(){
        subjectApi.getSubjectList().then(response => {
          //2获取所有的一级分类
          this.subjectOneList = response.data.list;
          //3遍历一级分类数组，比较当前一级courseInfo里的id是否相同
          for(var i = 0;i<this.subjectOneList.length;i++){
            //获取每个一级分类
            var oneSubject = this.subjectOneList[i];
            //比较当前一级分类id和courseInfo里的id是否相同
            if(this.courseInfo.subjectParentId === oneSubject.id){
              //获取一级分类下的二级分类
              this.subjectTwoList = oneSubject.children
            }
          }
        })
      },
      //点击一级分类触发显示对应二级分类 两级联动
      subjectLevelOneChanged(value) {
        //value：一级分类id值
        for (var i = 0; i < this.subjectOneList.length; i++) {
          //得到每个一级分类
          var oneSubject = this.subjectOneList[i];
          //判断所有一级分类id和点击一级分类是否一样
          if (value === oneSubject.id) {
            //从一级分类里获取其下的二级分类
            this.subjectTwoList = oneSubject.children;
            //选择其他一级分类时  清空当前所选二级分类
            this.courseInfo.subjectId = ''
          }
        }
      },
      //清空按钮的方法
      resetData() {
        //清空已输入数据
        this.courseQuery = {};
        //二级分类也要清除
        this.subjectTwoList=[];
        //查询所有  恢复未进行条件查询的状态
        this.getList()
      },

      //删除课程的方法
      removeDataById(courseId) {
        this.$confirm('此操作将永久删除该课程，以及该课程下的章节和视频，是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          courseApi.deleteCourseById(courseId)
            .then(response => { //删除成功
              //提示信息
              this.$message({
                type: 'success',
                message: '删除成功! 🈚'
              });
              //回到列表页面
              this.getList()
            })
        })
        //点取消执行catch方法  用户体验角度
        //此处无需进行提示取消
        //框架在./utils/request.js封装好了提示error的方法
      },
//导出excel表
      handleDownload() {
        this.downloadLoading = true;
        import('@/vendor/Export2Excel').then(excel => {
          const tHeader = [ '课程名称', '课时', '浏览量', '销售数量', '价格', '状态'];
          const filterVal = ['title', 'lessonNum', 'viewCount', 'buyCount', 'price', 'status'];
          const data = this.formatJson(filterVal);
          excel.export_json_to_excel({
            header: tHeader,
            data,
            filename: '心理课程列表'
          });
          this.downloadLoading = false
        });
        // 清空已输入数据
        this.psychologistQuery = {};
        // 查询所有  恢复未进行条件查询的状态
        this.getList()
      },
      formatJson(filterVal) {
        this.list.map(s => {
            if (s.status === 'Draft') {
              s.status = '未发布'
            } else {
              s.status = '发布'
            }
            return s
          }
        );
        return this.list.map(v => filterVal.map(j => {
          if (j === 'timestamp') {
            return parseTime(v[j])
          } else {
            return v[j]
          }
        }))
      }    }
  }
</script>
