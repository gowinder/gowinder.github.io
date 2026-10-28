# Adding a New Settings Page to chrome://settings in Chromium


The left-hand menu of the settings page is edited in

`chrome/browser/resources/settings/settings_menu/settings_menu.html`

If you want to add or remove a setting, that is where you change it.

To add one, put in a block like this:

    <a id="new_setting" href="/new_setting"
        hidden="[[!pageVisibility.new_setting]]">
      <iron-icon icon="settings:assignment"></iron-icon>
      $i18n{newSettingTitle}
    </a>

Then create a directory `chrome/browser/resources/settings/new_setting`

and put the settings page inside it.

Create `new_setting_page.html` and `new_setting_page.js` — that is the content.

Create `new_setting_section.htm` and `new_setting_section.js` — that is the section.

Create BUILD.gn

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

In `../basic_page/basic_page.html` add

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

Note that if you copied this straight over from another page, you have to register the name `settings-new_setting-page`, replacing the middle "new\_setting" with your own page name.

As for prefs="...", you need to add the corresponding pref name yourself, in `chrome/common/pref_names.h` / `.c`. The value path does not take a leading "prefs" — for example, if you add `new_setting.some_switcher`, then in the `html` page it becomes: `prefs="prefs.new_setting.some_switcher"`. Then register it in `chrome/browser/profiles/profile_impl.cc` → `ProfileImpl::RegisterProfilePrefs` (you can also write a dedicated ProfileImpl to manage it), then add a method in `chrome/browser/profile_resetter/profile_resetter.cc` to reset the property, or stuff it into another method (which is of course not great). In the header file, modify `Resettable` to add the `enum` value for the property page you need, and in `ProfileResetter::Reset` add the new value together with its corresponding function.

In `chrome/browser/prefs/chrome_pref_service_factory.cc` add the matching entry.

Important:

In `chrome/browser/extensions/api/settings_private/prefs_util.cc`, add the corresponding whitelist entry, otherwise opening the settings page throws: `”Prefs not found on element....“`

      // add by boost 2020-08-13 16:23:59
    (*s_whitelist)[::prefs::knew_settingSomeSwitch] =
        settings_api::PrefType::PREF_TYPE_BOOLEAN;

Adding the `i18n` text

Add a new ID in `chrome/app/settings_strings.grdp`.

In `chrome/browser/ui/webui/settings/settings_localized_strings_provider.cc`, add the variable that maps the `id` to the `webui`.

In `chrome/app/settings_strings.grdp` add the new `message`, for example:

    <message name="IDS_SETTINGS_new_setting_SETTINGS_TITLE" desc="The tile text of new_setting settings">
        some settings
      </message>

Then run `src\tools\grit>python`

    from grit.extern import tclib
    >>> tclib.GenerateMessageId("some settings");

You will get an output id, such as `49303453453534`. Use that output to add the matching translation in `src\chrome\app\resources\generated_resources_zh-CN.xtb`

    <translation id="49303453453534">一些设置</translation>

Adding the resource packaging: in `chrome/browser/resources/settings/settings_resources.grd`, you have to add one entry per file.

    <structure name="IDR_SETTINGS_new_setting_PAGE_HTML"
                     file="new_setting/new_setting_page.html"
                     type="chrome_html" />
          <structure name="IDR_SETTINGS_new_setting_PAGE_JS"
                     file="new_setting/new_setting_page.js"
                     type="chrome_html" />

Adding the visibility property, in `chrome/browser/resources/settings/page_visibility.js`

    *   new_setting: (boolean|undefined),

and add each one's initial value below it.

In `chrome/browser/resources/settings/route.js`

add the corresponding route

    *   new_setting: (undefined|!settings.Route)
    
    
    if (pageVisibility.new_setting != false) {
    
    r.NEW_SETTING = r.BASIC.createSection('/new_setting', 'new_setting');
    
    }

Add one line to `chrome/browser/resources/settings/BUILD.gn`

    group("closure_compile") {
      deps = [
        ":settings_resources",
        ....
        "new_setting_page:closure_compile",
      ]

