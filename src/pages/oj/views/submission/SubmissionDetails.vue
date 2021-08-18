<template>
  <Row type="flex" justify="space-around">
    <Col :span="20" id="status">
      <Alert v-if="status.type !== 'primary'" :type="status.type" show-icon>
        <span class="title">{{$t('m.' + status.statusName.replace(/ /g, "_"))}}</span>
        <div slot="desc" class="content">
          <template>
            <pre  v-if="isCE">{{submission.statistic_info.err_info}}</pre>
            <span v-if="submission.problem && submission.contest">{{$t('m.Problem')}}: <a :href="'/contest/' + submission.contest + '/problem/' + submission.problem" style="color: #495060;">{{submission.problem}}</a></span>
            <span v-else-if="submission.problem">{{$t('m.Problem')}}: <a :href='/problem/ + submission.problem' style="color: #495060;">{{submission.problem}}</a></span>
            <span>{{$t('m.Time')}}: {{submission.statistic_info.time_cost | submissionTime}}</span>
            <span>{{$t('m.Memory')}}: {{submission.statistic_info.memory_cost | submissionMemory}}</span>
            <span>{{$t('m.Lang')}}: {{submission.language}}</span>
            <span>{{$t('m.Author')}}: <a :href="'/user-home?username=' + submission.username" style="color: #495060;">{{submission.username}}</a></span>
          </template>
        </div>
      </Alert>
      <Alert v-else show-icon>
        <span class="title">{{$t('m.' + status.statusName.replace(/ /g, "_"))}}</span>
        <div slot="desc" class="content">
          <template>
            <pre  v-if="isCE">{{submission.statistic_info.err_info}}</pre>
            <span v-if="submission.problem && submission.contest">{{$t('m.Problem')}}: <a :href="'/contest/' + submission.contest + '/problem/' + submission.problem" style="color: #495060;">{{submission.problem}}</a></span>
            <span v-else-if="submission.problem">{{$t('m.Problem')}}: <a :href='/problem/ + submission.problem' style="color: #495060;">{{submission.problem}}</a></span>
            <span>{{$t('m.Time')}}: {{submission.statistic_info.time_cost | submissionTime}}</span>
            <span>{{$t('m.Memory')}}: {{submission.statistic_info.memory_cost | submissionMemory}}</span>
            <span>{{$t('m.Lang')}}: {{submission.language}}</span>
            <span>{{$t('m.Author')}}: <a :href="'/user-home?username=' + submission.username" style="color: #495060;">{{submission.username}}</a></span>
          </template>
        </div>
      </Alert>
      <Table v-if="submission.info && !isCE" :span="15" stripe :loading="loading" :disabled-hover="true" :columns="columns" :data="submission.info.data">
      </Table>
      <Highlight :code="submission.code" :language="submission.language" :border-color="status.color"></Highlight>
      <div v-if="submission.can_unshare" :span="15" id="share-btn">
        <Button v-if="submission.shared"
                type="warning" size="large" @click="shareSubmission(false)">
          {{$t('m.UnShare')}}
        </Button>
        <Button v-else
                type="primary" size="large" @click="shareSubmission(true)">
          {{$t('m.Share')}}
        </Button>
      </div>
    </Col>
  </Row>
</template>

<script>
  import api from '@oj/api'
  import {JUDGE_STATUS} from '@/utils/constants'
  import utils from '@/utils/utils'
  import Highlight from '@/pages/oj/components/Highlight'

  export default {
    name: 'submissionDetails',
    components: {
      Highlight
    },
    data () {
      return {
        columns: [
          {
            title: this.$i18n.t('m.ID'),
            align: 'center',
            type: 'index'
          },
          {
            title: this.$i18n.t('m.Status'),
            align: 'center',
            render: (h, params) => {
              return h('Tag', {
                props: {
                  color: JUDGE_STATUS[params.row.result].type
                }
              }, this.$i18n.t('m.' + JUDGE_STATUS[params.row.result].name.replace(/ /g, '_')))
            }
          },
          {
            title: this.$i18n.t('m.Memory'),
            align: 'center',
            render: (h, params) => {
              return h('span', utils.submissionMemoryFormat(params.row.memory))
            }
          },
          {
            title: this.$i18n.t('m.Time'),
            align: 'center',
            render: (h, params) => {
              return h('span', utils.submissionTimeFormat(params.row.cpu_time))
            }
          }
        ],
        submission: {
          result: '0',
          code: '',
          info: {
            data: []
          },
          statistic_info: {
            time_cost: '',
            memory_cost: ''
          }
        },
        isConcat: false,
        loading: false
      }
    },
    mounted () {
      this.getSubmission()
    },
    methods: {
      getSubmission () {
        this.loading = true
        api.getSubmission(this.$route.params.id).then(res => {
          this.loading = false
          let data = res.data.data
          if (data.info && data.info.data && !this.isConcat) {
            // score exist means the submission is OI problem submission
            if (data.info.data[0].score !== undefined) {
              this.isConcat = true
              const scoreColumn = {
                title: this.$i18n.t('m.Score'),
                align: 'center',
                key: 'score'
              }
              this.columns.push(scoreColumn)
              this.loadingTable = false
            }
            if (this.isAdminRole) {
              this.isConcat = true
              const adminColumn = [
                {
                  title: this.$i18n.t('m.Real_Time'),
                  align: 'center',
                  render: (h, params) => {
                    return h('span', utils.submissionTimeFormat(params.row.real_time))
                  }
                },
                {
                  title: this.$i18n.t('m.Signal'),
                  align: 'center',
                  key: 'signal'
                }
              ]
              this.columns = this.columns.concat(adminColumn)
            }
          }
          this.submission = data
          this.submission['problem'] = this.$route.query.problem
          this.submission['contest'] = this.$route.query.contest
        }, () => {
          this.loading = false
        })
      },
      shareSubmission (shared) {
        let data = {id: this.submission.id, shared: shared}
        api.updateSubmission(data).then(res => {
          this.getSubmission()
          if (shared) {
            this.$success('Người khác có thể xem lời giải của bạn!')
          } else {
            this.$success('Người khác không thể xem lời giải của bạn!')
          }
        }, () => {
        })
      }
    },
    computed: {
      status () {
        console.log(JUDGE_STATUS[this.submission.result])
        return {
          type: JUDGE_STATUS[this.submission.result].type,
          statusName: JUDGE_STATUS[this.submission.result].name,
          color: JUDGE_STATUS[this.submission.result].color
        }
      },
      isCE () {
        return this.submission.result === -2
      },
      isAdminRole () {
        return this.$store.getters.isAdminRole
      }
    }
  }
</script>

<style scoped lang="less">
  #status {
    .title {
      font-size: 20px;
    }
    .content {
      margin-top: 10px;
      font-size: 14px;
      span {
        margin-right: 10px;
      }
      pre {
        white-space: pre-wrap;
        word-wrap: break-word;
        word-break: break-all;
      }
    }
  }

  .admin-info {
    margin: 5px 0;
    &-content {
      font-size: 16px;
      padding: 10px;
    }
  }

  #share-btn {
    float: right;
    margin-top: 5px;
    margin-right: 0px;
  }

  pre {
    border: none;
    background: none;
  }
  .sticky {
    margin-top: 20px;
  }
  div.free-course > .title {
    background: green;
    text-transform: uppercase;
    color: white;
    margin: 0px;
    padding: 8px;
    font-weight: bold;
    font-size: 1.2em;
  }
  div.free-course > ul{
    border: solid 4px green;
    padding: 10px;
    margin-top: 0px;
  }
  div.free-course > ul li::before {
    content: "📚";
    margin: 0px 10px 0 0px;
  }
  div.free-course > ul > li {
    font-size: 1.2em;
    padding-top: 10px;
    list-style: none;
  }
  div.free-course > ul > li > a {
    text-decoration:none;
    color: inherit;
    // font-family: monospace, sans-serif;
  }
</style>
