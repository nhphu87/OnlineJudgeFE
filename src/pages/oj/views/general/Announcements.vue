<template>
  <Row type="flex" :gutter="18">
    <Col :span="containerSpan">
      <Panel shadow :padding="10" >
        <div slot="title">
          {{title}}
        </div>
        <div slot="extra">
          <Button v-if="listVisible" type="info" @click="init" :loading="btnLoading"><span class="ivu-icon ivu-icon-refresh"></span> {{$t('m.Refresh')}}</Button>
          <Button v-else icon="ios-undo" @click="goBack">{{$t('m.Back')}}</Button>
        </div>
        <transition-group name="announcement-animate" mode="in-out">
          <div class="no-announcement" v-if="!announcements.length" key="no-announcement">
            <p>{{$t('m.No_Announcements')}}</p>
          </div>
          <template v-if="listVisible">
            <ul class="announcements-container" key="list">
              <li v-for="announcement in announcements" :key="announcement.title">
                <div class="flex-container">
                  <div class="title"><a class="entry" @click="goAnnouncement(announcement)">
                    {{announcement.title}}</a></div>
                  <div class="date">{{announcement.create_time | localtime }}</div>
                  <div class="creator"> {{$t('m.By')}} {{announcement.created_by.username}}</div>
                </div>
              </li>
            </ul>
            <Pagination v-if="!isContest"
                        key="page"
                        :total="total"
                        :page-size="limit"
                        @on-change="getAnnouncementList">
            </Pagination>
          </template>

          <template v-else>
          <div v-katex v-html="announcement.content" key="content" class="content-container markdown-body"></div>
          </template>
        </transition-group>
      </Panel>
      <Row v-if="!isContest" type="flex" :gutter="10" style="margin-top: 70px;">
            <Col  :span="12">
              <Panel shadow style="padding-top: 10px; padding-bottom: 10px;">
                <div slot="title">Bài tập mới</div>
                <ul style="margin-left: 40px;margin-bottom: 20px;">
                  <li style="padding: 5px 0px;"  v-for="p in problemList" :key="p.id">
                    <a class="link-style" :href="'/problem/' + p._id">{{p._id}} - {{p.title}}</a>
                  </li>
                </ul>
              </Panel>
            </Col>
            <Col  :span="12">
              <Panel shadow style="padding-top: 10px;padding-bottom: 10px;">
                <div slot="title">{{$t('m.Ranklist_Title')}}</div>
                <ol style="margin-left: 40px;margin-bottom: 20px;">
                   <li v-for="u in dataRank" :key="u.id" style="padding: 5px 0px">
                      <a :style="'font-weight: 600;color: ' + u.color" :href="'/user-home?username=' + u.user.username"
                         :title=" u.title + ' ' + u.user.username">
                      {{u.user.username}}
                      </a> - {{u.accepted_number}} bài
                   </li>
                </ol>
              </Panel>
            </Col>
        </Row>
    </Col>
    <Col :span="5" v-if="!isContest" >
      <Panel shadow>
        <div style="font-size:16px; text-align:center; width:100%; line-height:16px; background: transparent; color:#636e72;">Hôm nay là</div>
        <div class="today">
          <div class="nowWeek">{{nowWeek}}</div>
          <div class="nowDate">
            {{nowDate}}
          </div>
        </div>
        <div v-if="days" style="margin:0 auto; margin-bottom:15px; font-size:12px; text-align:center; width:160px; line-height:16px; background: transparent; color:#636e72;">Bạn đã có <strong>{{days}} </strong> chuỗi ngày học</div>
        <div style="margin-top:-10px; margin:0 auto; font-size:14px; text-align:center; width:80%; line-height:16px; background: transparent; color:#636e72;">{{word}}</div>
        <Button v-if="!SighinStatus" type="primary" icon="ios-alarm" @click="Sighin" long style="margin-top:20px; margin-bottom:20px; margin-left:10%; width:80%;">Điểm danh</Button>
        <Button v-else type="primary" icon="ios-alarm" long disabled style="margin-top:20px; margin-bottom:20px; margin-left:10%; width:80%;">
            Điểm danh
        </Button>
      </Panel>
      <Panel shadow style="margin-top: 37px;padding-bottom: 5px;">
        <div slot="title" style="margin-left: -10px;margin-bottom: -10px;">{{$t('m.Similar_Site')}}</div>
        <ul style="margin-left: 40px;margin-bottom: 20px;">
          <li style="padding: 5px 0px;"><a href="#" class="link-style" onclick="event.preventDefault();window.open('https://freecontest.net/?ref=nhpoj', '_blank');">Free Contest (🇻🇳)</a></li>
          <li style="padding: 5px 0px;"><a href="#" class="link-style" onclick="event.preventDefault();window.open('https://oj.vnoi.info/?ref=nhpoj', '_blank');">VNOJ: VNOI Online Judge (🇻🇳)</a></li>
          <li style="padding: 5px 0px;"><a href="#" class="link-style" onclick="event.preventDefault();window.open('http://ntucoder.net/?ref=nhpoj', '_blank');">NTUCoder (🇻🇳)</a></li>
          <li style="padding: 5px 0px;"><a href="#" class="link-style" onclick="event.preventDefault();window.open('https://codeforces.com/?ref=nhpoj', '_blank');">Codeforces (🇬🇧)</a></li>
        </ul>
      </Panel>
    </Col>
  </Row>
</template>

<script>
  import api from '@oj/api'
  // import axios from 'axios'
  import Pagination from '@oj/components/Pagination'
  import { RULE_TYPE, USER_GRADE } from '@/utils/constants'
  import { mapState } from 'vuex'

  export default {
    name: 'Announcement',
    components: {
      Pagination
    },
    data () {
      return {
        limit: 10,
        total: 10,
        problemList: [],
        problemLimit: 10,
        dataRank: [],
        rankLimit: 10,
        query: {
          keyword: '',
          difficulty: '',
          tag: '',
          page: 1,
          orderby: '-create_time'
        },
        btnLoading: false,
        announcements: [],
        announcement: '',
        listVisible: true,
        timer: null,
        containerSpan: 19,
        SighinStatus: false,
        nowWeek: '',
        nowDate: '',
        word: '',
        days: 0
      }
    },
    mounted () {
      this.init()
      this.timer = setInterval(() => {
        this.setNowTimes()
      }, 1000)
      this.getWord()
      this.isSighin()
      this.getProblemList()
      this.getRankData()
    },
    methods: {
      init () {
        if (this.isContest) {
          this.containerSpan = 24
          this.getContestAnnouncementList()
        } else {
          this.getAnnouncementList()
        }
      },
      getRankData () {
        api.getUserRank(0, this.rankLimit, RULE_TYPE.ACM).then(res => {
          this.dataRank = res.data.data.results
          for (let i in this.dataRank) {
            this.dataRank[i]['color'] = USER_GRADE[this.dataRank[i].grade].color
            this.dataRank[i]['title'] = USER_GRADE[this.dataRank[i].grade].name
          }
        }).catch(() => {
        })
      },
      getProblemList () {
        let offset = 0
        api.getProblemList(offset, this.problemLimit, this.query).then(res => {
          this.problemList = res.data.data.results
        })
      },
      getAnnouncementList (page = 1) {
        this.btnLoading = true
        api.getAnnouncementList((page - 1) * this.limit, this.limit).then(res => {
          this.btnLoading = false
          this.announcements = res.data.data.results
          this.total = res.data.data.total
        }, () => {
          this.btnLoading = false
        })
      },
      getContestAnnouncementList () {
        this.btnLoading = true
        api.getContestAnnouncementList(this.$route.params.contestID).then(res => {
          this.btnLoading = false
          this.announcements = res.data.data
        }, () => {
          this.btnLoading = false
        })
      },
      goAnnouncement (announcement) {
        this.announcement = announcement
        this.listVisible = false
      },
      goBack () {
        this.listVisible = true
        this.announcement = ''
      },
      getWord () {
        // axios.get('http://nghoangphu.name.vn/danhngon').then(response => {
        //   this.word = response.data.hitokoto
        // })
        this.word = '- Học tập là gian khổ, muốn không khổ thì đừng gian  -'
      },
      setNowTimes () {
        let myDate = new Date()
        let weeks = ['Chủ nhật', 'Thứ 2', 'Thứ 3', 'Thứ 4', 'Thứ 5', 'Thứ 6', 'Thứ 7']
        let mouth = ['01', '02', '03', '04', '05', '06', '07', '08', '09', '10', '11', '12']
        this.nowDate = String(myDate.getDate() < 10 ? '0' + myDate.getDate() : myDate.getDate()) + '/' + mouth[myDate.getMonth()] + '/' + myDate.getFullYear()
        this.nowWeek = weeks[myDate.getDay()]
      },
      isSighin () {
        api.GetSighinStatus().then(res => {
          if (res.data.data === null || res.data.data.sighinstatus === 'false') {
            this.SighinStatus = false
          } else {
            this.SighinStatus = true
          }
          this.days = res.data.data !== null ? res.data.data.continue_sighin_days : 0
        })
      },
      Sighin () {
        api.UserSighin().then(res => {
          if (res.data.data === 'Singined') {
            this.$Notice.error({
              title: 'Điểm danh không thành công',
              desc: 'Bạn đã điểm danh hôm nay rồi, hãy quay lại vào ngày mai nhé'
            })
            this.isSighin()
          } else {
            this.$Notice.success({
              title: 'Điểm danh thành công',
              desc: 'Chúc mừng bạn có thêm ' + (res.data.data.experience) + ' điểm chuyên cần, nhớ quay lại vào ngày mai nhé.'
            })
            this.days += 1
            this.SighinStatus = true
          }
        })
      }
    },
    computed: {
      ...mapState(['website']),
      title () {
        if (this.listVisible) {
          return this.isContest ? this.$i18n.t('m.Contest_Announcements') : this.$i18n.t('m.Announcements')
        } else {
          return this.announcement.title
        }
      },
      isContest () {
        return !!this.$route.params.contestID
      }
    }
  }
</script>

<style scoped lang="less">
  .announcements-container {
    margin-top: -10px;
    margin-bottom: 10px;
    li {
      padding-top: 15px;
      list-style: none;
      padding-bottom: 15px;
      margin-left: 20px;
      font-size: 16px;
      border-bottom: 1px solid rgba(187, 187, 187, 0.5);
      &:last-child {
        border-bottom: none;
      }
      .flex-container {
        .title {
          flex: 1 1;
          text-align: left;
          padding-left: 10px;
          a.entry {
            color: #495060;
            &:hover {
              color: #2d8cf0;
              border-bottom: 1px solid #2d8cf0;
            }
          }
        }
        .creator {
          flex: none;
          width: 200px;
          text-align: center;
        }
        .date {
          flex: none;
          width: 200px;
          text-align: center;
        }
      }
    }
  }

  .content-container {
    padding: 0 20px 20px 20px;
  }

  .no-announcement {
    text-align: center;
    font-size: 16px;
  }changeLocale

  .announcement-animate-enter-active {
    color: rgba(255, 255, 255, 0.5);
    animation: fadeIn 1s;
  }
  .nowWeek {
    text-align: center;
    padding-top: 20px;
    font-size: 25px;
    font-weight: 600;
  }
  .nowDate {
    text-align: center;
    padding-bottom: 10px;
    font-size: 30px;
    font-weight: 600;
  }
  .tag-btn {
    margin-right: 10px;
    margin-bottom: 20px;
  }
  .tag-btn:hover {
    a {
       color: #2d8cf0;
    }
  }
  .link-style {
    color:#264c67
  }
  .link-style:hover {
    color: #2d8cf0;
  }
</style>
