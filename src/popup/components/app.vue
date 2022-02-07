<template>
	<div class="popup_page">
		<p class="popup_title">UMEM助手</p>

    <select
      v-model="currToken"
      @change="onSelectChange($event)"
      class="popup_select"
      placeholder="服务器"
    >
      <option
        v-for="(options,domain) in domainList"
        :key="domain"
        :value="options.token"
        class="popup_option"
      >
        {{options.domain}}
      </option>
    </select><br>

    <input
      v-model="currPort"
      class="popup_input"
      type="number"
      placeholder="端口号，默认8000"
    />

		<button
      @click="getCookie()"
      class="popup_button">{{bt_txt}}
    </button>

    <p class="popup_tip" v-if="tipTxt.length>0">{{tipTxt}}</p>

	</div>
</template>

<script lang="ts">
import { defineComponent, reactive, toRefs } from 'vue'
import axios from 'axios';
export default defineComponent({
  created() {
    this.init()
  },
  setup() {
    const state = reactive({
      bt_txt: '一键更新配置',

      tipTxt: '',

      form: {
        uc_user_agent: '',
        uc_token: '',
        uc_token_haitang: '',
        uc_cookie: ''
      },

      currToken: '',
      currDomain: '',
      currPort: '8000',
      domainList: [
        {
          domain: '',
          token: ''
        }
      ]
    })

    /**
     * 初始化token
     */
    const init = () => {
      // 初始化
      state.form.uc_user_agent = navigator.userAgent

      // 获取token
      chrome.cookies.getAll({name: 'UMEM_TOKEN'}, (cookies) => {
        state.domainList = []
        for (const c of cookies) {
          state.domainList.push(
            {
              domain: c.domain,
              token: c.value
            }
          )
        }
        if (state.domainList.length > 0) {
          state.currDomain = state.domainList[0].domain
          state.currToken = state.domainList[0].token
        }
        console.log(`初始化成功：${state.domainList}`)
      })
    }

    /**
     * 获取cookie
     */
    const getCookie = () => {
      if (!state.domainList || state.domainList.length === 0) {
        state.tipTxt = '未检测到UMEM页面'
        return
      }
      if (!state.currDomain || state.currDomain.length === 0) {
        state.tipTxt = '请先选择目标页面地址'
        return
      }
      if (!state.currToken || state.currToken.length === 0) {
        state.tipTxt = `请在 ${getCurrUrl()} 中登录UMEM后再操作`
        return
      }

      state.tipTxt = ''
      chrome.cookies.getAll({
        domain: 'umeng.com'
      }, (cookies) => {
        for (const c of cookies) {
          if (c.name.search('XSRF-TOKEN') !== -1 && c.domain !== 'mobile.umeng.com') {
            // 过滤掉cookies中不符合要求的token
            continue
          } else {
            state.form.uc_cookie = `${state.form.uc_cookie}${c.name}=${c.value};`
          }
        }
        state.form.uc_cookie.split(';').forEach((item) => {
          if (item.search('XSRF-TOKEN-HAITANG') !== -1 && state.form.uc_token_haitang.length === 0) {
            state.form.uc_token_haitang = item.split('=')[1]
          } else if (item.search('XSRF-TOKEN') !== -1 && state.form.uc_token.length === 0) {
            state.form.uc_token = item.split('=')[1]
          }
        });
        uploadInfo();
      })
    }

    /**
     * 上传信息
     */
    function uploadInfo() {
      const url = `${getCurrUrl()}/api/save_config`
      axios.post(url, state.form, { headers: {'Authorization': state.currToken} })
        .then(function (res) {
          console.log(res);
          if (res.data.code === 200) {
            state.tipTxt = '更新成功'
            chrome.tabs.query({title: 'UMEM-*'}, (tabs) => {
              for (const tab of tabs) {
                chrome.tabs.reload(tab.id)
              }
            })
          } else {
            state.tipTxt = `${getCurrUrl()} ${res.data.msg}：${res.data.code}`
          }
        })
        .catch(function (e) {
          console.log(e);
          state.tipTxt = `${getCurrUrl()} 更新失败: ${e.toString()}`
        });
    }

    /**
     * 选择项改变时的监听
     * @param event
     */
    const onSelectChange = (event) => {
      const i = event.target.selectedIndex
      if (state.domainList.length > i) {
        state.currDomain = state.domainList[i].domain
      }
    }

    /**
     * 获取当前的服务链接信息
     */
    const getCurrUrl = () => {
      return `http://${state.currDomain}:${state.currPort}`
    }

    return {
      ...toRefs(state),
      getCookie,
      init,
      onSelectChange
    }
  }
})

</script>

<style lang="less" scoped>
	.popup_page{
    padding: 0px 20px 20px 20px;
    font-size: 16px;
    width: auto;
		color: black;

    .popup_title{
      text-align: center;
    }

	}

	.popup_select{
    width: 180px;
    text-align: center;
		color: black;
    height: 30px;
    padding: 0px 10px;
    margin: 0px 10px 0px 10px;
	}

	.popup_option{
    width: 180px;
    text-align: center;
		color: black;
    height: 30px;
    padding: 0px 10px;
    margin-top: 40px;
	}

	.popup_input{
    width: 160px;
    text-align: center;
		color: black;
    height: 30px;
    margin: 10px 10px 0px 10px;
    background-color: #e6e6e6;
    border: none;
    border-radius: 4px;
    padding: 0px 10px;
	}

	.popup_button{
    width: 180px;
    text-align: center;
		color: white;
    height: 30px;
    margin: 10px 10px 0px 10px;
    background-color: #409EFF;
    border: none;
    border-radius: 4px;
    padding: 0px 10px;
	}

	.popup_tip{
    width: 180px;
    text-align: center;
    font-size: 12px;
		color: #999999;
    height: auto;
    margin: 10px 10px 0px 10px;
	}
</style>
