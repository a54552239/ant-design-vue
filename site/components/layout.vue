<script>
import AllDemo from '../demo'
import Header from './header'
import zhCN from 'antd/locale-provider/zh_CN'
import enUS from 'antd/locale-provider/default'
import sortBy from 'lodash/sortBy'
import { isZhCN } from '../util'
import { Provider, create } from '../../components/_util/store'
import NProgress from 'nprogress'
import debounce from 'lodash/debounce'
import addDOMEventListener from 'add-dom-event-listener'

const docsList = [
  { key: 'introduce', enTitle: 'Ant Design of Vue', title: 'Ant Design of Vue' },
  { key: 'getting-started', enTitle: 'Getting Started', title: '快速上手' },
  { key: 'use-with-vue-cli', enTitle: 'Use in vue-cli', title: '在 vue-cli 中使用' },
  { key: 'customize-theme', enTitle: 'Customize Theme', title: '定制主题' },
  { key: 'changelog', enTitle: 'Change Log', title: '更新日志' },
  { key: 'i18n', enTitle: 'Internationalization', title: '国际化' },
]
function getOffsetTop (element, container = window) {
  if (!element) {
    return 0
  }

  if (!element.getClientRects().length) {
    return 0
  }

  const rect = element.getBoundingClientRect()

  if (rect.width || rect.height) {
    if (container === window) {
      container = element.ownerDocument.documentElement
      return rect.top - container.clientTop
    }
    return rect.top - container.getBoundingClientRect().top
  }

  return rect.top
}

export default {
  props: {
    name: String,
    showDemo: Boolean,
    showApi: Boolean,
  },
  data () {
    this.store = create({
      currentSubMenu: [],
    })
    this.subscribe()
    return {
      showSideBars: true,
      currentSubMenu: [],
      sidebarHeight: document.documentElement.offsetHeight,
    }
  },
  provide () {
    return {
      demoContext: this,
    }
  },
  beforeDestroy () {
    if (this.unsubscribe) {
      this.unsubscribe()
    }
    clearTimeout(this.timer)
    if (this.resizeEvent) {
      this.resizeEvent.remove()
    }
    if (this.debouncedResize && this.debouncedResize.cancel) {
      this.debouncedResize.cancel()
    }
  },
  mounted () {
    this.$nextTick(() => {
      this.addSubMenu()
      const nprogressHiddenStyle = document.getElementById('nprogress-style')
      if (nprogressHiddenStyle) {
        this.timer = setTimeout(() => {
          nprogressHiddenStyle.parentNode.removeChild(nprogressHiddenStyle)
        }, 0)
      }

      this.updateHeight()
      this.debouncedResize = debounce(() => {
        this.updateHeight()
      }, 200)
      this.resizeEvent = addDOMEventListener(window, 'resize', this.debouncedResize)
    })
  },
  watch: {
    '$route.path': function () {
      this.store.setState({ currentSubMenu: [] })
      this.addSubMenu()
    },
  },
  methods: {
    updateHeight () {
      const el = this.$refs.sidebar.$el
      const offsetTop = getOffsetTop(el)
      const docHeight = document.documentElement.offsetHeight
      this.sidebarHeight = docHeight - offsetTop
    },
    addSubMenu () {
      if (this.$route.path.indexOf('/docs/vue/') !== -1) {
        this.$nextTick(() => {
          const menus = []
          const doms = [...this.$refs.doc.querySelectorAll(['h2', 'h3'])]
          doms.forEach(dom => {
            const id = dom.id
            if (id) {
              const title = dom.textContent.split('#')[0].trim()
              menus.push({ cnTitle: title, usTitle: title, id })
            }
          })
          this.currentSubMenu = menus
        })
      }
    },
    subscribe () {
      const { store } = this
      this.unsubscribe = store.subscribe(() => {
        this.currentSubMenu = this.store.getState().currentSubMenu
      })
    },
    getSubMenu (isCN) {
      const currentSubMenu = this.currentSubMenu
      const lis = []
      currentSubMenu.forEach(({ cnTitle, usTitle, id }) => {
        const title = isCN ? cnTitle : usTitle
        lis.push(<a-anchor-link href={`#${id}`} title={title} />)
      })
      const showApi = this.$route.path.indexOf('/components/') !== -1
      return (
        <a-anchor offsetTop={70} class='demo-anchor'>
          {lis}
          {showApi ? <a-anchor-link title='API' href='#API' /> : ''}
        </a-anchor>
      )
    },
    getDocsMenu (isCN) {
      const docsMenu = []
      docsList.forEach(({ key, enTitle, title }) => {
        const k = isCN ? `${key}-cn` : key
        docsMenu.push(<a-menu-item key={k}>
          <router-link to={`/ant-design-vue/docs/vue/${k}/`}>{isCN ? title : enTitle }</router-link>
        </a-menu-item>)
      })
      return docsMenu
    },
    resetDocumentTitle (component, name, isCN) {
      let titleStr = 'Vue Antd'
      if (component) {
        const { subtitle, title } = component
        const componentName = isCN ? subtitle + ' ' + title : title
        titleStr = componentName + ' - ' + titleStr
      } else {
        const currentKey = docsList.filter((item) => {
          return item.key === name
        })
        if (currentKey.length) {
          titleStr = (isCN ? currentKey[0]['title'] : currentKey[0]['enTitle']) + ' - ' + titleStr
        }
      }
      document.title = titleStr
    },
    mountedCallback () {
      NProgress.done()
      document.documentElement.scrollTop = 0
    },
  },

  render () {
    const name = this.name
    const isCN = isZhCN(name)
    // name = name.replace(/-cn\/?$/, '')
    const titleMap = {}
    const menuConfig = {
      General: [],
      Layout: [],
      Navigation: [],
      'Data Entry': [],
      'Data Display': [],
      Feedback: [],
      Other: [],
    }
    const searchData = []
    for (const [title, d] of Object.entries(AllDemo)) {
      const type = d.type || 'Other'
      const key = `${title.replace(/(\B[A-Z])/g, '-$1').toLowerCase()}`
      titleMap[key] = title
      AllDemo[title].key = key
      menuConfig[type] = menuConfig[type] || []
      menuConfig[type].push(d)
    }
    const reName = name.replace(/-cn\/?$/, '')
    // const Demo = new Vue({
    //   template: '<demo-component/>',
    //   components: {
    //     'demo-component': () => import(`../../components/${AllDemo[titleMap[reName]].key}/demo/index.vue`),
    //   },
    // })
    // AllDemo[titleMap[reName]]
    const MenuGroup = []
    for (const [type, menus] of Object.entries(menuConfig)) {
      const MenuItems = []
      sortBy(menus, ['title']).forEach(({ title, subtitle }) => {
        const linkValue = isCN
          ? [<span>{title}</span>, <span class='chinese'>{subtitle}</span>]
          : [<span>{title}</span>]
        let key = `${title.replace(/(\B[A-Z])/g, '-$1').toLowerCase()}`
        if (isCN) {
          key = `${key}-cn`
        }
        searchData.push({
          title,
          subtitle,
          url: `/ant-design-vue/components/${key}/`,
        })
        MenuItems.push(<a-menu-item key={key}>
          <router-link to={`/ant-design-vue/components/${key}/`}>{linkValue}</router-link>
        </a-menu-item>)
      })
      MenuGroup.push(<a-menu-item-group title={type}>{MenuItems}</a-menu-item-group>)
    }
    let locale = zhCN
    if (!isCN) {
      locale = enUS
    }
    const config = AllDemo[titleMap[reName]]
    this.resetDocumentTitle(config, reName, isCN)
    const { showSideBars, sidebarHeight } = this
    return (
      <div class='page-wrapper'>
        <Header searchData={searchData} name={name}/>
        <a-locale-provider locale={locale}>
          <div class='main-wrapper'>
            <a-row>
              <a-col v-show={showSideBars} style={{ height: `${sidebarHeight}px` }} ref='sidebar' class='site-sidebar' xxl={4} xl={5} lg={5} md={6} sm={8} xs={12}>
                <div class='drawer-mask' onClick={() => { this.showSideBars = false }}></div>
                <a-menu

                  class='aside-container menu-site'
                  selectedKeys={[name]}
                  defaultOpenKeys={['Components']}
                  inlineIndent={40}
                  mode='inline'>
                  {this.getDocsMenu(isCN)}
                  <a-sub-menu title={`Components(${searchData.length})`} key='Components'>
                    {MenuGroup}
                  </a-sub-menu>
                </a-menu>
                <div class='close-drawer' onClick={() => { this.showSideBars = false }}>
                  <a-icon type='close'/>
                </div>
              </a-col>
              <div v-show={!showSideBars} class='open-drawer' onClick={() => { this.showSideBars = true }}>
                <a-icon type='bars'/>
              </div>
              <a-col style='float: right;' xxl={20} xl={19} lg={19} md={18} sm={24} xs={24}>
                <div class='content main-container'>
                  <div class='toc-affix' style='width: 120px;'>
                    {this.getSubMenu(isCN)}
                  </div>
                  {this.showDemo ? <Provider store={this.store} key={isCN ? 'cn' : 'en'}>
                    <router-view
                      class={`demo-cols-${config.cols || 2}`}
                      {...{ directives: [
                        {
                          name: 'mountedCallback',
                          value: this.mountedCallback,
                        },
                      ] }}
                    ></router-view>
                  </Provider> : ''}
                  {this.showApi ? <div class='markdown api-container' ref='doc'>
                    <router-view
                      {...{ directives: [
                        {
                          name: 'mountedCallback',
                          value: this.mountedCallback,
                        },
                      ] }}
                    ></router-view>
                  </div> : ''}
                </div>
              </a-col>
            </a-row>

          </div>
        </a-locale-provider>

        { name.indexOf('back-top') === -1 ? <a-back-top /> : null }
      </div>
    )
  },
}
</script>
