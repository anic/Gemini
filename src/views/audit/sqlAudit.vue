<style lang="less" scoped>
  @import '../../styles/common.less';
  @import '../../styles/table.less';

  .demo-Circle-custom {
    & h1 {
      color: #3f414d;
      font-size: 28px;
      font-weight: normal;
    }

    & p {
      color: #657180;
      font-size: 14px;
      margin: 10px 0 15px;
    }

    & span {
      display: block;
      padding-top: 15px;
      color: #657180;
      font-size: 14px;

      &:before {
        content: '';
        display: block;
        width: 50px;
        height: 1px;
        margin: 0 auto;
        background: #e0e3e6;
        position: relative;
        top: -15px;
      }
    ;
    }

    & span i {
      font-style: normal;
      color: #3f414d;
    }
  }

  p {
    word-wrap: break-word;
    word-break: break-all;
    overflow: hidden;
  }
</style>
<template>
  <div>
    <Row>
      <Card>
        <p slot="title">
          <Icon type="md-person"></Icon>
          工单审核
        </p>
        <Row>
          <Col span="24">
            <Form inline>
              <FormItem>
                <Poptip trigger="hover" title="提示" content="此开关用于打开实时表格数据更新功能">
                  <i-switch v-model="valve" @on-change="refreshForm" size="large">
                    <span slot="open">打开</span>
                    <span slot="close">关闭</span>
                  </i-switch>
                </Poptip>
              </FormItem>
              <FormItem>
                <Input placeholder="账号名" v-model="find.user" @on-keyup.enter="queryData"></Input>
              </FormItem>
              <FormItem>
                <DatePicker format="yyyy-MM-dd HH:mm" type="datetimerange" placeholder="请选择查询的时间范围"
                            v-model="find.picker" @on-change="find.picker=$event" style="width: 250px"
                            :editable="false"></DatePicker>
              </FormItem>
              <FormItem>
                <Button type="success" @click="queryData">查询</Button>
                <Button type="primary" @click="queryCancel" class="margin-left-10">重置</Button>
              </FormItem>
            </Form>
            <Table border :columns="columns" :data="tableData" stripe>
              <template slot-scope="{ row }" slot="action">
                <div v-if="row.Status !== 5 || auth==='perform'">
                  <Button type="primary" @click="openOrder(row)" size="small" ghost
                          v-if="row.Status === 2 || auth==='perform' && row.Status === 5">审批
                  </Button>
                  <Button type="success" @click="orderDetail(row)" v-if="row.Status === 1 || row.Status === 4"
                          size="small" ghost>
                    执行结果
                  </Button>
                  <Poptip
                          confirm
                          title="确定要中止该工单吗？"
                          @on-ok="delayKill(row)"
                          transfer>
                    <Button type="error" v-if="row.Status === 3 && row.Delay !== 'none'"
                            size="small" ghost>
                      延时工单中止
                    </Button>
                  </Poptip>

                  <Button ghost size="small" class="margin-left-10" @click="timerOsc(row)"
                          v-if="row.Status === 3 && row.Type === 0" type="warning">osc进度
                  </Button>
                </div>
              </template>
              <template slot-scope="{ row }" slot="delay">
                <span v-if="row.Delay !== 'none'">{{row.Delay}}</span>
                <span v-else>无</span>
              </template>
            </Table>
            <br>
            <Page :total="pagenumber" show-elevator @on-change="refreshData" :page-size="20" ref="page"></Page>
          </Col>
        </Row>
      </Card>
    </Row>


    <Modal
            v-model="osc.open"
            title="OSC进度展示" :closable="false"
            @on-cancel="oscClose"
            @on-ok="oscKill"
            ok-text="osc终止"
    >
      <Col offset="6">
        <i-circle
                :size="250"
                :trail-width="4"
                :stroke-width="5"
                :percent="osc.percent"
                stroke-linecap="square"
                stroke-color="#43a3fb">
          <div class="demo-Circle-custom">
            <h1>{{osc.percent}}%</h1>
            <span>
                当前正在执行第
                <i>{{osc.current}}</i>
              条
            </span>
          </div>
        </i-circle>
      </Col>
    </Modal>

    <Modal v-model="modal2" width="1000">
      <p slot="header" style="color:#f60;font-size: 16px">
        <Icon type="information-circled"></Icon>
        <span>SQL工单详细信息</span>
      </p>
      <Form label-position="right">
        <FormItem label="环境:">
          <span>{{ formitem.IDC }}</span>
        </FormItem>
        <FormItem label="连接名称:">
          <span>{{ formitem.Source }}</span>
        </FormItem>
        <FormItem label="数据库库名:">
          <span>{{ formitem.Base }}</span>
        </FormItem>
        <FormItem label="数据库表名:">
          <span>{{ formitem.Table }}</span>
        </FormItem>
        <FormItem label="定时执行:">
          <span>{{ formitem.Delay }}</span>
        </FormItem>
        <FormItem label="工单说明:">
          <span>{{ formitem.Text }}</span>
        </FormItem>
        <FormItem>
          <Table :columns="sql_columns" :data="sql" :max-height="300"></Table>
        </FormItem>
        <FormItem label="选择执行人:" v-if="multi && auth === 'admin'" required>
          <Select v-model="multi_name" style="width: 20%">
            <Option v-for="i in multi_list" :value="i.Username" :key="i.Username">{{i.Username}}</Option>
          </Select>
        </FormItem>
      </Form>

      <div slot="footer">
        <Button @click="modal2 = false">取消</Button>
        <Button type="warning" @click.native="testTo()" :loading="loading">
          <span v-if="!loading">检测sql</span>
          <span v-else>检测中</span></Button>
        <template v-if="multi">
          <Button type="error" @click="rejectTo()">驳回</Button>
          <Button type="success" @click="agreedTo()" :disabled="summit" v-if="auth === 'admin'">同意</Button>
          <Button type="success" @click="performTo(false)" v-else-if="auth === 'perform'">执行</Button>
        </template>
        <template v-else>
          <Button type="error" @click="rejectTo()">驳回</Button>
          <Button type="success" @click="performTo(false)" :disabled="summit">执行</Button>
        </template>
      </div>
    </Modal>

    <Modal v-model="reject.reje" @on-ok="rejectText">
      <p slot="header" style="color:#f60;font-size: 16px">
        <Icon type="information-circled"></Icon>
        <span>SQL工单驳回理由说明</span>
      </p>
      <Input v-model="reject.textarea" type="textarea" :autosize="{minRows: 15,maxRows: 15}"
             placeholder="请填写驳回说明"></Input>
    </Modal>
  </div>
</template>
<script>
    import axios from 'axios'
    import expandRow from '../../components/expandTable.vue';
    export default {
        name: 'Sqltable',
        data() {
            return {
                osc: {
                    percent: 0,
                    open: false,
                    current: 1
                },
                loading: false,
                sql_columns: [
                    {
                        type: 'expand',
                        width: 50,
                        render: (h, params) => {
                            return h(expandRow, {
                                props: {
                                    row: params.row
                                }
                            })
                        }
                    },
                    {
                        title: '当前检查的sql',
                        key: 'SQL',
                        render: (h, params) => {
                            let text = params.row.SQL.substring(0, 40) + '...';
                            return  h('span', text)
                        }

                    },
                    {
                        title: '阶段',
                        key: 'Status',
                        width: '150'
                    },
                    {
                        title: '错误等级',
                        key: 'Level',
                        width: '100'
                    },
                    {
                        title: '错误信息',
                        key: 'Error',
                        tooltip: true
                    },
                    {
                        title: '影响行数',
                        key: 'AffectRows',
                        width: '120'
                    }
                ],
                columns: [
                    {
                        title: '工单编号:',
                        key: 'WorkId',
                        sortable: true,
                        sortType: 'desc',
                        width: 155
                    },
                    {
                        title: '工单说明:',
                        key: 'Text',
                        tooltip: true
                    },
                    {
                        title: '是否备份',
                        key: 'Backup',
                        width: 100
                    },
                    {
                        title: '提交时间:',
                        key: 'Date',
                        sortable: true
                    },
                    {
                        title: '提交账号',
                        key: 'Username',
                        sortable: true
                    },
                    {
                        title: '真实姓名',
                        key: 'RealName',
                        sortable: true
                    },
                    {
                        title: '定时执行',
                        key: 'Delay',
                        slot: 'delay'
                    },
                    {
                        title: '执行人',
                        key: 'Executor',
                        sortable: true
                    },
                    {
                        title: '状态',
                        key: 'Status',
                        width: 150,
                        render: (h, params) => {
                            const row = params.row;
                            let color = '';
                            let text = '';
                            if (row.Status === 2) {
                                color = 'primary';
                                text = '待审核'
                            } else if (row.Status === 0) {
                                color = 'error';
                                text = '驳回'
                            } else if (row.Status === 1) {
                                color = 'success';
                                text = '执行成功'
                            } else if (row.Status === 4) {
                                color = 'error';
                                text = '执行失败'
                            } else if (row.Status === 5) {
                                color = 'default';
                                text = '待执行'
                            } else {
                                color = 'warning';
                                text = '执行中'
                            }
                            return h('Tag', {
                                props: {
                                    type: 'dot',
                                    color: color
                                }
                            }, text)
                        },
                        sortable: true
                    },
                    {
                        title: '操作',
                        key: 'action',
                        width: 200,
                        align: 'center',
                        slot: 'action'
                    }
                ],
                modal2: false,
                sql: [],
                gen: 0, 
                formitem: {},
                summit: true,
                reject: {
                    reje: false,
                    textarea: ''
                },
                tableData: [],
                pagenumber: 1,
                callback_time: null,
                switch_show: true,
                multi: Boolean,
                auth: sessionStorage.getItem('auth'),
                multi_list: {},
                multi_name: '',
                reboot: null,
                valve: true,
                find: {
                    picker: [],
                    user: '',
                    valve: false
                },
                ws: null,
                fk: ''
            }
        },
        methods: {
            delayKill(vl) {
                axios.get(`${this.$config.url}/audit/kill/${vl.WorkId}`)
                    .then(res => {
                        this.$config.notice(res.data);
                        this.refreshData()
                    })
                    .catch(err => this.$config.err_notice(this, err))
            },
            timerOsc(vl) {
                this.osc.open = true;
                this.osc.current = 0;
                this.osc.percent = 0;
                this.fk = vl.WorkId;
                let vm = this;
                this.ws = setInterval(function () {
                    vm.openOSC(vl)
                }, 3000)
            },
            openOSC() {
                axios.get(`${this.$config.url}/audit/fetch_osc/${this.fk}`)
                    .then(res => {
                        this.osc.percent = res.data.p;
                        this.osc.current = res.data.s;
                    })
                    .catch(err => this.$config.err_notice(this, err))
            },
            oscKill() {
                axios.delete(`${this.$config.url}/audit/fetch_osc/${this.fk}`)
                    .then(res => {
                        this.$config.notice(res.data);
                        this.oscClose()
                    })
                    .catch(err => {
                        this.$config.err_notice(this, err);
                        this.oscClose()
                    })
            },
            oscClose() {
                clearInterval(this.ws)
            },
            openOrder(row) {
                this.summit = true;
                this.sql = [];
                this.modal2 = true
                row.Status === 2 ? this.switch_show = true : this.switch_show = false
                axios.get(`${this.$config.url}/audit/sql?k=${row.WorkId}`)
                    .then(res => {
                        this.sql = res.data.sql;
                        this.formitem.IDC = res.data.idc;
                        this.formitem = {
                            WorkId: row.WorkId,
                            IDC: res.data.idc,
                            Source: res.data.source,
                            Delay: res.data.delay,
                            Base: res.data.base,
                            Text: res.data.text,
                            Table: res.data.table,
                            Type: res.data.type
                        }
                    })
                    .catch(err => {
                        this.$config.err_notice(this, err)
                    })
            },
            agreedTo() {
                if (this.multi_name === '') {
                    this.$Message.error('请选择执行人!')
                } else {
                    axios.post(`${this.$config.url}/audit/refer/perform`, {
                        'perform': this.multi_name,
                        'WorkId': this.formitem.WorkId
                    })
                        .then(res => {
                            this.$config.notice(res.data)
                            this.modal2 = false;
                            this.refreshData(this.$refs.page.currentPage)
                        })
                        .catch(error => {
                            this.$config.err_notice(this, error)
                        })
                }
            },
            performTo(ignoreErrors) {
                if (this.gen > 0 && !ignoreErrors) {
                  this.$config.confirm(
                    this,
                    "提交的SQL语句不符合规范，是否继续执行",
                    () => this.performTo(true)
                  );
                  return;
                }

                this.modal2 = false;
                axios.post(`${this.$config.url}/audit/execute`, {
                    'workid': this.formitem.WorkId
                })
                    .then(res => {
                        this.$config.notice(res.data);
                        this.refreshData(this.$refs.page.currentPage)
                    })
                    .catch(error => {
                        this.$config.err_notice(this, error)
                    })
            },
            rejectTo() {
                this.modal2 = false;
                this.reject.reje = true
            },
            testTo() {
                this.loading = true;
                let s = '';
                for (let i of this.sql) {
                    s += i.SQL
                }
                let isDML = false;
                if (this.formitem.Type === 1) {
                    isDML = true
                }
                axios.put(`${this.$config.url}/fetch/test`, {
                    'source': this.formitem.Source,
                    'database': this.formitem.Base,
                    'table': this.formitem.table,
                    'sql': s,
                    'isDMl': isDML
                })
                    .then(res => {
                        this.sql = res.data;
                        this.gen = 0;
                        this.sql.forEach(vl => {
                            if (vl.Level !== 0) {
                                this.gen += 1
                            }
                        });
                        if (this.gen === 0) {
                            this.summit = false
                        } else {
                            this.summit = true
                        }
                        //始终允许提交，在审核时进行确认
                        this.summit = false
                        this.loading = false
                    })
                    .catch(error => {
                        this.$config.err_notice(this, error)
                    })
            },
            rejectText() {
                axios.post(`${this.$config.url}/audit/reject`, {
                    'text': this.reject.textarea,
                    'work': this.formitem.WorkId
                })
                    .then(res => {
                        this.$config.notice(res.data);
                        this.refreshData(this.$refs.page.currentPage)
                    })
                    .catch(error => {
                        this.$config.err_notice(this, error)
                    })
            },
            orderDetail(row) {
                this.$router.push({
                    name: 'orderlist',
                    query: {workid: row.WorkId, status: row.Status}
                })
            },
            refreshData(vl = 1) {
                axios.put(`${this.$config.url}/audit`, {
                    page: vl,
                    find: this.find
                })
                    .then(res => {
                        this.multi = res.data.multi;
                        if (!this.multi) {
                            for (let i = 0; i < this.columns.length; i++) {
                                if (this.columns[i].key === 'Executor') {
                                    this.columns.splice(i, 1)
                                }
                            }
                        }
                        this.tableData = res.data.data;
                        this.tableData.forEach((item) => {
                            (item.Backup === 1) ? item.Backup = '是' : item.Backup = '否'
                        });
                        this.pagenumber = res.data.page;
                        this.multi_list = res.data.multi_list
                    })
                    .catch(error => {
                        this.$config.err_notice(this, error)
                    })
            },
            refreshForm(vl) {
                if (vl) {
                    let vm = this;
                    this.reboot = setInterval(function () {
                        vm.refreshData(vm.$refs.page.currentPage)
                    }, 5000)
                } else {
                    clearInterval(this.reboot)
                }
            },
            queryData() {
                this.find.valve = true
                this.refreshData()
            },
            queryCancel() {
                this.find = this.$config.clearPicker(this.find);
                this.$refs.page.currentPage = 1;
                this.refreshData()
            }
        },
        mounted() {
            this.refreshData();
            this.refreshForm(this.valve)
        },
        destroyed() {
            clearInterval(this.ws);
            clearInterval(this.reboot)
        }
    }
</script>
<!-- remove delete request -->
