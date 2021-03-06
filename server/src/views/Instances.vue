<template>
  <a-layout v-if="pageInit">
    <a-layout-header></a-layout-header>
    <a-layout-content v-if="instances.length">
      <a-row type="flex">
        <a-card
          hoverable
          style="width: 300px; margin: 10px"
          v-for="instance in instances"
          :key="instance.Name"
        >
          <template class="ant-card-actions" #actions>
            <template v-if="instance.status == 'offline'">
              <a-tooltip title="实例配置修改" placement="bottom">
                <setting-outlined @click="configInstance(instance)" />
              </a-tooltip>
              <a-tooltip title="实例依赖更新" placement="bottom">
                <CloudDownloadOutlined @click="updateInstance(instance)" />
              </a-tooltip>
              <a-tooltip title="实例删除" placement="bottom">
                <a-popconfirm
                  :title="`是否删除该实例？`"
                  ok-text="Yes"
                  cancel-text="No"
                  @confirm="removeInstance(instance)"
                >
                  <DeleteOutlined />
                </a-popconfirm>
              </a-tooltip>
            </template>
            <template v-else-if="instance.status == 'online'">
              <a-tooltip title="实例操作面板" placement="bottom">
                <DesktopOutlined
                  @click="$router.push('/dashboard/' + instance.Name)"
                />
              </a-tooltip>
              <a-popconfirm
                title="是否重启实例进程?"
                ok-text="Yes"
                cancel-text="No"
                @confirm="restartInstance(instance)"
              >
                <a-tooltip title="重启实例" placement="bottom">
                  <ReloadOutlined />
                </a-tooltip>
              </a-popconfirm>
            </template>
          </template>
          <a-card-meta :description="'📁' + instance.Path"
            ><template #title>
              <div
                :class="
                  instance.status == 'offline' ? 'status-red' : 'status-green'
                "
              ></div>
              <span>{{ instance.Name }}</span>
              <span v-if="instance.status == 'online'"
                >[{{ instance.pid }}]</span
              >
              <a-tooltip title="实例启动开关" placement="top">
                <a-switch
                  @change="
                    (checked) =>
                      checked ? startInstance(instance) : killInstance(instance)
                  "
                  :loading="false"
                  :checked="instance.status == 'online'"
                  size="small"
                  style="float: right"
                />
              </a-tooltip>
            </template>
          </a-card-meta>
        </a-card>
      </a-row>
    </a-layout-content>
    <a-empty
      v-if="instances.length == 0"
      style="margintop: 20vh; font-size: 20px"
      description="你还没有实例哦，快点击下面按钮，去创建实例吧！"
    >
      <a-button
        size="large"
        style="margintop: 30px; font-size: 20px"
        @click="$router.push('/create')"
        >创建新实例</a-button
      >
    </a-empty>
    <a-layout-footer v-if="instances.length">
      <a-button @click="$router.push('/create')">创建新实例</a-button>
    </a-layout-footer>
    <a-modal
      :title="currentInstance.Name"
      v-model:visible="visible.config"
      :confirm-loading="confirmLoading"
      @ok="modifyConfig"
    >
      <a-tabs
        v-if="currentInstance.config"
        tab-position="left"
        :style="{ height: '300px' }"
      >
        <a-tab-pane
          v-for="(plugin, name) in currentInstance.config"
          :key="name"
          :tab="name"
        >
          <a-form
            :model="plugin"
            :labelCol="{ span: 8 }"
            :wrapperCol="{ span: 12 }"
          >
            <a-form-item v-for="(v, k) in plugin" :label="k" :key="k">
              <a-input
                v-if="typeof v == 'string'"
                v-model:value="plugin[k]"
              ></a-input>
              <a-input-number
                v-else-if="typeof v == 'number'"
                v-model:value="plugin[k]"
              />
              <a-switch
                v-else-if="typeof v == 'boolean'"
                v-model:checked="plugin[k]"
              />
              <template v-else-if="Array.isArray(v)">
                <div>{{ JSON.stringify(v) }}</div>
                <div class="array-item" v-for="(v1, index) in plugin[k]" :key="k + index">
                  <a-input v-model:value="plugin[k][index]">
                  </a-input>
                  <MinusCircleOutlined
                    class="dynamic-delete-button"
                    @click="removeDomain(index, k, name)"
                  />
                </div>
                <a-button type="link" @click="v.push('')">+</a-button>
              </template>
            </a-form-item>
          </a-form>
        </a-tab-pane>
      </a-tabs>
    </a-modal>
    <a-modal
      :title="currentInstance.Name"
      v-model:visible="visible.update"
      @ok="visible.update = false"
    >
      <pre v-html="updateLog" style="max-height: 50vh"></pre>
      <span v-if="updateLog.search('timeout') > -1">
        检测到更新依赖出现超时，请查看更新日志内容，确保成功更新依赖 <br/>
        提示：出现超时，一般和 GOPROXY 代理设置有关，请修改代理设置，并使配置生效，然后再重新运行monica命令
      </span>
      <span v-if="updateLog.search('downloading') < 0">
        当前依赖版本已经是最新的啦😊
      </span>
    </a-modal>
  </a-layout>
</template>
<script>
import { reactive, ref, onUnmounted, getCurrentInstance } from 'vue'

import fastrx from 'fastrx'
const rx = fastrx.rx
import {
  SettingOutlined,
  ReloadOutlined,
  PoweroffOutlined,
  DeleteOutlined,
  PlayCircleOutlined,
  LoadingOutlined,
  FileAddOutlined,
  DesktopOutlined,
  CloudDownloadOutlined,
  MinusCircleOutlined
} from '@ant-design/icons-vue'
import { notification } from 'ant-design-vue'
export default {
  components: {
    SettingOutlined,
    ReloadOutlined,
    PoweroffOutlined,
    DeleteOutlined,
    PlayCircleOutlined,
    LoadingOutlined,
    FileAddOutlined,
    DesktopOutlined,
    CloudDownloadOutlined,
    MinusCircleOutlined
  },
  setup() {
    const instances = ref([])
    // 页面初始化
    const pageInit = ref(false)

    const unmountedOb = rx.fromLifeHook(onUnmounted)
    const {
      ctx: { $message }
    } = getCurrentInstance()

    rx.interval(5000)
      .startWith(0)
      .switchMap(() =>
        rx
          .fromFetch('/api/instance/list')
          .switchMap((x) => rx.fromPromise(x.json()))
      )
      .takeUntil(unmountedOb)
      .subscribe((x) => {
        instances.value = x
        pageInit.value = true
      })
    const visible = reactive({
      config: false,
      update: false,
      create: false
    })
    const confirmLoading = ref(false)
    const commonRes = (msg) => (response) => {
      if (response.ok) {
        $message.success(msg)
      } else {
        $message.error(response.statusText)
      }
    }
    const updateLog = ref('')
    function startInstance(instance) {
      instance.status = ''
      fetch('/api/instance/start?name=' + instance.Name, {
        method: 'POST'
      })
        .then((response) => response.json())
        .then((data) => {
          // code 是 1 代表失败
          if (data.code != 0) {
            let text = ''
            if (data.code == 1)
              text =
                '解决方案：权限不足，需要你采用 root 身份重新启动 monica，然后再次尝试'
            if (data.code == 2)
              text =
                '解决方案：该实例某些插件端口和其他正在运行实例插件端口有冲突，需要你进行端口调整，避免启动的插件有端口重复'
            notification['error']({
              message: '创建实例失败',
              description: '失败提示：' + data.msg,
              onClick: () => {
                console.log('Notification Clicked!')
              },
              // 加一行文字
              btn: text
            })
          } else $message.success(data.msg)
        })
    }
    const result = {
      updateLog,
      confirmLoading,
      currentInstance: {},
      instances,
      visible,
      pageInit,
      configInstance(instance) {
        result.currentInstance = instance
        visible.config = true
      },
      modifyConfig() {
        confirmLoading.value = true
        fetch(
          '/api/instance/config/modify?name=' + result.currentInstance.Name,
          {
            method: 'POST',
            body: JSON.stringify(result.currentInstance.config),
            headers: {
              'content-type': 'application/json'
            }
          }
        ).then((response) => {
          confirmLoading.value = false
          if (response.ok) {
            visible.config = false
            $message.success('修改配置文件成功')
          } else {
            $message.error(response.statusText)
          }
        })
      },
      killInstance(instance) {
        fetch('/api/instance/kill?name=' + instance.Name, {
          method: 'POST'
        }).then(commonRes('已关闭实例'))
      },
      startInstance(instance) {
        instance.status = ''
        fetch('/api/instance/start?name=' + instance.Name, {
          method: 'POST'
        })
          .then((response) => {
            return response.json()
          })
          .then((data) => {
            // code 是 1 代表失败
            if (data.code != 0) {
              let text = ''
              if (data.code == 1)
                text =
                  '解决方案：权限不足，需要你采用 root 身份重新启动 monica，然后再次尝试'
              if (data.code == 2)
                text =
                  '解决方案：该实例某些插件端口和其他正在运行实例插件端口有冲突，需要你进行端口调整，避免启动的插件有端口重复'
              notification['error']({
                message: '创建实例失败',
                duration: 7,
                description: '失败提示：' + data.msg,
                onClick: () => {
                  console.log('Notification Clicked!')
                },
                // 加一行文字
                btn: text
              })
            } else {
              $message.success('创建实例成功')
              notification['success']({
                message: '创建实例成功',
                duration: 7,
                description: '成功提示：' + data.msg
              })
            }
          })
      },
      restartInstance(instance) {
        instance.status = ''
        fetch('/api/instance/start?name=' + instance.Name, {
          method: 'POST'
        })
          .then((response) => {
            return response.json()
          })
          .then((data) => {
            // code 是 1 代表失败
            if (data.code != 0) {
              let text = ''
              if (data.code == 1)
                text =
                  '解决方案：权限不足，需要你采用 root 身份重新启动 monica，然后再次尝试'
              if (data.code == 2)
                text =
                  '解决方案：该实例某些插件端口和其他正在运行实例插件端口有冲突，需要你进行端口调整，避免启动的插件有端口重复'
              notification['error']({
                message: '创建实例失败',
                duration: 7,
                description: '失败提示：' + data.msg,
                onClick: () => {
                  console.log('Notification Clicked!')
                },
                btn: text
              })
            } else {
              notification['success']({
                message: '已重启实例',
                duration: 7,
                description: '成功提示：' + data.msg
              })
            }
          })
      },
      removeInstance(instance) {
        instance.status = ''
        fetch('/api/instance/remove?name=' + instance.Name, {
          method: 'DELETE'
        }).then(commonRes('实例已删除'))
      },
      updateInstance(instance) {
        visible.update = true
        result.currentInstance = instance
        updateLog.value = ''
        rx.fromEventSource('/api/instance/update?name=' + instance.Name)
          .takeUntil(unmountedOb)
          .subscribe(
            (msg) => {
              updateLog.value += msg + '\n'
            },
            (e) => e.target.close(),
            () => {}
          )
      },
      removeDomain(index, k, name) {
        console.log(index,k,name)
        result.currentInstance.config[name][k].splice(index, 1);
      }
    }
    return result
  }
}
</script>

<style>
.ant-notification-topRight {
  width: 40vw;
}
.array-item {
  position: relative;
}
.array-item .dynamic-delete-button {
  cursor: pointer;
  position: absolute;
  right: -40px;
  top: 6px;
  font-size: 24px;
  color: #999;
  transition: all 0.3s;
}
.array-item .dynamic-delete-button:hover {
  color: #777;
}
.array-item .dynamic-delete-button[disabled] {
  cursor: not-allowed;
  opacity: 0.5;
}
.ant-modal {
  width: 70vw !important;
}

pre {
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>
