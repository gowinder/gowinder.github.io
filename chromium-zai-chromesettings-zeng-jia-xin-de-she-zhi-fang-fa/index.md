# chromium 在chrome://settings增加新的设置方法


设置左边栏目菜单修改位置在

`chrome/browser/resources/settings/settings_menu/settings_menu.html`

如果需要增加或者去掉设置就在里面改

增加的话，加一段：

    <a id="new_setting" href="/new_setting"
        hidden="[[!pageVisibility.new_setting]]">
      <iron-icon icon="settings:assignment"></iron-icon>
      $i18n{newSettingTitle}
    </a>

然后增加 一个目录 `chrome/browser/resources/settings/new_setting`

在里面放设置的页面

新建 `new_setting_page.html` 和`new_setting_page.js`, 这个是内容

新建 `new_setting_section.htm`, `new_setting_section.js` 这个是区块

新建 BUILD.gn

    # Copyright 2018 The Chromium Authors. All rights reserved.
    # Use of this source code is governed by a BSD-style license that can be
    # found in the LICENSE file.
    
    import("//third_party/closure_compiler/compile_js.gni")
    
    js_type_check("closure_compile") {
      deps = [
      ]
    }
    
    js_library("new_setting_page") {
      deps = [
        ":new_setting_section",
        "..:open_window_proxy",
        "..:route",
        "../prefs:prefs_behavior",
        "../settings_page:settings_animated_pages",
        "//ui/webui/resources/js:assert",
        "//ui/webui/resources/js:cr",
      ]
      externs_list = [
        "$externs_path/passwords_private.js",
        "$externs_path/settings_private.js",
      ]
    }
    
    js_library("new_setting_section") {
      deps = [
        ":address_edit_dialog",
        "//ui/webui/resources/cr_elements/cr_action_menu:cr_action_menu",
        "//ui/webui/resources/js:assert",
        "//ui/webui/resources/js:load_time_data",
        "//ui/webui/resources/js/cr/ui:focus_without_ink",
      ]
      externs_list = [  ]
    }

`../basic_page/basic_page.html`增加

    <link rel="import" href="../new_setting_page/new_setting_page.html">
    
    <template is="dom-if" if="[[showPage_(pageVisibility.new_setting)]]"
                          restamp>
                    <settings-section page-title="$i18n{newSettingTitle}"
                                      section="new_setting">
                        <settings-new_setting-page prefs="{{prefs}}"
                                                page-visibility="[[pageVisibility]]">
                        </settings-new_setting-page>
                    </settings-section>
                </template>

注意，如果是直接从其它页面拷贝过来的，要注册 `settings-new_setting-page` 这个名字，中间"new\_setting"改成对应自己的页面名字

其中 prefs="..." ,需要自己增加对应设置名称，在`chrome/common/pref_names.h` / `.c`中增加，值路径不需要前面加“prefs",如，增加一个`new_setting.some_switcher`,那么在`html`页面中，就对应：`prefs="prefs.new_setting.some_switcher"`，然后在 `chrome/browser/profiles/profile_impl.cc` → `ProfileImpl::RegisterProfilePrefs` 中注册(也可以自己专门写个ProfileImpl来管理），然后 在`chrome/browser/profile_resetter/profile_resetter.cc` 中增加一个方法来重置属性，或者放到别的方法里面（当然不好）,头文件中修改 `Resettable`，增加需要的属性页`enum`值,`ProfileResetter::Reset`中，将新加的值和对应的函数加进去

`chrome/browser/prefs/chrome_pref_service_factory.cc` 增加对应条目

重要：

在 `chrome/browser/extensions/api/settings_private/prefs_util.cc` 中，增加对应的白名单，不然打开设置页面会报错：`”Prefs not found on element....“`

      // add by boost 2020-08-13 16:23:59
    (*s_whitelist)[::prefs::knew_settingSomeSwitch] =
        settings_api::PrefType::PREF_TYPE_BOOLEAN;

增加文字`i18n`

在`chrome/app/settings_strings.grdp` 中增加一个新的ID

在`chrome/browser/ui/webui/settings/settings_localized_strings_provider.cc` 中，增加对应`id`到`webui`的对照变量

在`chrome/app/settings_strings.grdp` 中增加新的`message`,如：

    <message name="IDS_SETTINGS_new_setting_SETTINGS_TITLE" desc="The tile text of new_setting settings">
        some settings
      </message>

然后 运行 `src\tools\grit>python`

    from grit.extern import tclib
    >>> tclib.GenerateMessageId("some settings");

会得到一个输出id，如`49303453453534`，使用这个输出，在`src\chrome\app\resources\generated_resources_zh-CN.xtb` 中增加对应翻译

    <translation id="49303453453534">一些设置</translation>

增加资源打包，在`chrome/browser/resources/settings/settings_resources.grd`中，有几个文件就得加几个

    <structure name="IDR_SETTINGS_new_setting_PAGE_HTML"
                     file="new_setting/new_setting_page.html"
                     type="chrome_html" />
          <structure name="IDR_SETTINGS_new_setting_PAGE_JS"
                     file="new_setting/new_setting_page.js"
                     type="chrome_html" />

增加 显示属性， 在`chrome/browser/resources/settings/page_visibility.js`

    *   new_setting: (boolean|undefined),

并在下面增加各自的初始值

在`chrome/browser/resources/settings/route.js`

增加对应路由

    *   new_setting: (undefined|!settings.Route)
    
    
    if (pageVisibility.new_setting != false) {
    
    r.NEW_SETTING = r.BASIC.createSection('/new_setting', 'new_setting');
    
    }

`chrome/browser/resources/settings/BUILD.gn` 增加一行

    group("closure_compile") {
      deps = [
        ":settings_resources",
        ....
        "new_setting_page:closure_compile",
      ]

