# 全文検索画面ー絞り込み条件を複数表示対応
---

### 1. 絞り込み条件の階層表示「現状」と「改修要望」
- **現状**：
  - 階層１：グループ名
    - 階層２：タイプ名
      - 階層３：検索結果コンテンツの一覧
- **改修要望**：
  - 階層１：グループ名
    - 階層２：タイプ名
      ```diff
      +⦿ 階層３：タイプ名（サブ）
         - 階層４：検索結果コンテンツの一覧
      ```

### 2. 絞り込み条件の階層表示仕組み

- `コンテンツタイプ`は「`$`」で紐づけて,テーブル「`contents_type_pattern`」に保存しています。
   - 例：`グループ/ニュース`のタイプ：`custom_sj$news`.


      | pattern_id | pattern_name | type_field_name |
      |------------|--------------|-----------------|
      | 4          | ニュース         | custom_sj$news  |
      | 5          | 通達           | custom_sj$info  |
      | 6          | 規定集、マニュアル    | custom_sj$rule  |
      | 7          | その他          | custom_sj$etc   |
      | 2          | 掲示板、HP       | custom_sj$stock |

- `コンテンツタイプ`と`コンテンツ`所属するアプリケーションは`imfr_ut_gp_cmn_frm_00033`に保存しています。例：
   | imfr_ud_txt_application_id | imfr_ud_txt_table_name | imfr_ud_rbt_type_pattern |
   |----------------------------|------------------------|--------------------------|
   | bb_afo2140b                | gp_hp_5stp_t_info      | 掲示板、HP                   |
   | hp_abb1320b                | gp_hp_5stp_t_info      | 通達                       |
   | hp_abb1321b                | gp_hp_5stp_t_info      | 規定集、マニュアル                |
- コンテツテーブル`imfr_ut_gp_cmn_frm_00033`とコンテツ・タイプ・パターンマスタ`contents_type_pattern`は`パターン名`で紐づけています。

- グループとタイプ、及び`親子関係`は`contentssearch-template-config_custom_sj.xml`に定義しています。
   - 例：
   ```xml
    <template-page type="custom_sj" sort-key="9">
      <type-display-key>CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_CUSTOM_SJ</type-display-key>
    </template-page>
    <template-page type="stock" sort-key="2">
      <parent-type>custom_sj</parent-type>
      <type-display-key>CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_STOCK</type-display-key>
      <template-path>im_contents_search/template/custom_sj/custom_sj_message_template.jssp</template-path>
    </template-page>
    ... 
   ```

- グループとタイプの表示名は`conf\message\im_contents_search_custom_sj_message.properties`に定義しています。
   - 例：
   ```properties
    #結果画面：共通
    CAP.Z.SNP.CONTENTSSEARCH.COMMON.TEMPLATE.UPDATE.DATE=日付
    CAP.Z.SNP.CONTENTSSEARCH.COMMON.TEMPLATE.CONTENTS.TYPE=コンテンツ種別
    #コンテンツ種別
    CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_CUSTOM_SJ=グループ
    CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_STOCK=掲示板、HP
    CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_NEWS=ニュース
    CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_INFO=通達
    CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_RULE=規程集、マニュアル
    CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_ETC=その他
   ```
- クローラは、ジョブスケジューラのパラメータから`グループ番号`を取得し、`コンテンツタイプ（TYPE定数）`と組み合わせてSolrに送って、インデックスID、コンテンツタイプと検索結果の紐づけ関係ファイルを生成します。
- 全文検索画面で、検索キーワードをsolrへ送信後、solrからコンテンツタイプとアプリケーションの紐づけ結果を返し、画面に表示します。

### 3. 改修要望に応じる対策例：
1. テーブル「`contents_type_pattern.type_field_name`のタイプ定義を追加する. 例：  

      | pattern_id | pattern_name | type_field_name |
      |------------|--------------|-----------------|
      | 8         | 通達サブ  | custom_sj`$`info`$`infosub |
      | 9         | ニュースサブ  | custom_sj`$`news`$`newssub |
2. グループとタイプ`親子関係`を`contentssearch-template-config_custom_sj.xml`に追加する。 例：

   ```xml
    ... 
    <template-page type="other_a" sort-key="2">
      <parent-type>etc</parent-type>
      <type-display-key>CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_INFO_SUB</type-display-key>
      <template-path>im_contents_search/template/custom_sj/custom_sj_message_template.jssp</template-path>
    </template-page>
    ... 
   ```
3. 表示名を`conf\message\im_contents_search_custom_sj_message.properties`に追加する. 例：
   ```diff
   ...
   + #CAP.Z.IWP.CONTENTSSEARCH.CUSTOM_SJ.MESSAGE.CONTENTS.TYPE_INFO_SUB=通達サブ
   ```
   :warning: message.properitesに定義漏れの場合、`type_field_name`に定義している`id`(例：`infosub`)を表示する.
 
4. `ScratchCrawler.java`にコンテンツタイプの追加ロジックを加える.
   - `contents_type_pattern.type_field_name`のタイプ定義を取得する.　例: custom_sj`$`info`$`infosub
   - `$`で階層を分割する
   - 階層付きのタイプ文字列を作成し、IMコンテンツオブジェクトに追加する. 
     - 例：`custom_sj`,`custom_sj$info`,`custom_sj$info$infosub`の３つタイプを`InputContent`に追加する.
5. 対応後：検索結果の画面表示 
   - ![image](/uploads/d2b93b53a535ee4499118e1a92c59abf/image.png)

### 3. 補足
- 既存の絞り込み条件表示階層タイプを変更する場合、
  - `contents_type_pattern`と`imfr_ut_gp_cmn_frm_00033`の関連データを変更した後、
  - ジョブネットにてインデックス再作成が必要です。
- 参考資料： 
  - 検索結果テンプレート設定: https://document.intra-mart.jp/library/iap/public/configuration/im_configuration_reference/texts/im_contents_search/contentssearch-template-config/index.html