  <template>
  <Card>
    <h1>关注课程</h1>
    <br>
    <Form :label-width="80" :model="query" inline>
      <FormItem label="课程名字：" prop="lesson_name">
        <AutoComplete style="width: 180px" v-model="query.lesson_name" placeholder="请输入用户名字">
          <Option v-for="d in data" :value="d.lesson_name" :key="d.lesson_name">{{ d.lesson_name }}</Option>
        </AutoComplete>
      </FormItem>
      <FormItem label="学期：" prop="term">
        <Select v-model="query.term" style="width:200px">
          <Option v-for="item in terms" :value="item.name" :key="item.name">{{ item.name }}</Option>
        </Select>
      </FormItem>
      <FormItem >
        <Button type="primary" @click="onSearch(query)">查询</Button>
        </FormItem>
      <FormItem >
        <Button type="primary" @click="onSearch(query)">导入</Button>
      </FormItem>
    </Form>

    <LessonProfileModal
      :show="showLessonProfileModal"
      @onOK="onProfileModalOK"
      @onCancel="onProfileModalCancel"
      :lesson_id="this.selected_lesson_id"
    ></LessonProfileModal>

    <BatchLessonWatchModal
      :show="showBatchLessonWatchModal"
      @onOK="onBatchRemoveModalOK"
      @onCancel="onBatchRemoveModalCancel"
    ></BatchLessonWatchModal>

    <Table  @on-selection-change="selectLessons" border stripe :columns="columns" :data="data"></Table>
      <div style="float: right;">
        <Page :total="total" show-total :page-size="pages._per_page" :current="pages._page" @on-change="onPageChange"></Page>
      </div>
    <FloatBar><Button type="error" @click="onBatchWatchClick">批量取消关注</Button>
    </FloatBar>
  </Card>
</template>

<script>
import LessonProfileModal from './components/LessonProfileModal'
import BatchLessonRemoveModal from './components/BatchLessonWatchModal'
import {queryNoticeLessons, putLesson} from '../../service/api/lesson'
import {queryTerms, getCurrentTerms} from '../../service/api/term'
import FloatBar from '_c/float_bar/float_bar'
import {updateWithinField} from 'Libs/tools'

export default {
  components: {LessonProfileModal, FloatBar, BatchLessonWatchModal: BatchLessonRemoveModal},
  data: function () {
    return {

      query: {
        lesson_name:undefined,
        term: undefined,
      }, // 查询用的参数
      total: 0, // 总数量
      data: [], // 数据
      terms: [],
      selected_lesson_ids: [],
      selected_lesson_id: '', // 选中编辑的课程ids
      showLessonProfileModal: false, // 展示编辑弹窗
      showBatchLessonWatchModal: false,
      pages: {
        _page: 1,
        _per_page: 10
      }, // 分页
      columns: [
        {
          type: 'selection',
          width: 60,
          align: 'center'
        },
        {
          title: '课程名字',
          render: function (h, params) {
            return (
              <span>{ params.row.lesson_name }</span>
            )
          }
        },
        {
          title: '课程属性',
          render: function (h, params) {
            return (
              <span>{ params.row.lesson_attribute }</span>
            )
          }
        },

        {
          title: '分配组别',
          render: function (h, params) {
            return (
              <span>{ params.row.assign_group }</span>
          )
          }
        },
        {
          title: '关注原因',
          render: function (h, params) {
            return (
              <span>{ params.row.notice_reason }</span>
          )
          }
        },
        {
          title: '课程状态',
          render: (h, params) => {
            if (params.row.lesson_state === '未完成'){
              return h('Tag', { props: {color:"red"}}, params.row.lesson_state)
            } else {
              return h('Tag', { props: {color:"blue"}}, params.row.lesson_state)
            }
          }
        },
        {
          title: '操作',
          align: 'center',
          render: (h, params) => {
            return h('div', [
              h('Button', {
                props: {
                  type: 'primary',
                  size: 'small'
                },
                style: {
                  marginRight: '2px'
                },
                on: {
                  click: () => {
                    this.selected_lesson_id = params.row.lesson_id
                    this.showLessonProfileModal = true
                  }
                }
              }, '查看')
            ])
          }
        }
      ]
    }
  },
  methods: {
    onTableChange (query, pages) {
      // 数据表发生变化请求数据
      let args = {...query, ...pages}
      queryNoticeLessons(args).then((resp) => {
        this.selected_lesson_ids = []
        this.data = resp.data.notice_lessons
        this.total = resp.data.total
        this.$router.push({path: '/lesson/notice_lesson', query: {...args, ...this.query}})
      })
    },
    onPageChange (page) {
      // 分页变化
      this.pages._page = page
      this.onTableChange(this.query, this.pages)
    },
    onSearch () {
      // 查询变化
      this.pages._page = 1
      this.onTableChange(this.query, this.pages)
    },
    onProfileModalOK (lesson) {
      // 更新框确定 关闭
      putLesson(lesson).then((resp) => {
        this.showLessonProfileModal = false
      })
    },
    onProfileModalCancel () {
      this.showLessonProfileModal = false
    },
    onBatchRemoveModalOK (lesson) {
      this.showBatchLessonWatchModal = false
    },
    onBatchRemoveModalCancel () {
      this.showBatchLessonWatchModal = false
    },
    selectLessons: function (selection) {
      // 批量选择触发
      this.selected_lesson_ids = selection.map((item) => {
        return item.id
      })
    },
    onBatchWatchClick: function () {
      // 批量关注触发
      this.showBatchLessonWatchModal = true
      console.log('selected lessons id : ', this.selected_lesson_ids)
    }
  },
  mounted: function () {
    const args = this.$route.query
    updateWithinField(this.query, args)
    updateWithinField(this.pages, args)
    queryTerms().then((resp) => {
      this.terms = resp.data.terms
    })
    getCurrentTerms().then((termResp) => {
      this.query.term = termResp.data.term.name
      queryNoticeLessons  ({ ...this.pages, ...this.query}).then((resp) => {
        this.data = resp.data.notice_lessons
        this.total = resp.data.total
        this.$router.push({path: '/lesson/notice_lesson', query: {...this.query, ...this.pages}})
      })
    })
  }
}
</script>

<style scoped>

</style>
