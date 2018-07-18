---
layout:     post
title:      View-Layout注入框架
subtitle:   简洁视图注入
date:       2018-03-22
author:     Gavin
header-img: img/post-bg-coffee.jpg
catalog: true
tags:
    - android
    - 依赖注入
    - 解耦
---

### 框架开发前
在没有这套框架之前，我们在activity、fragment、自定义view、listview的viewholder甚至你能想的更多。
大概是这样：

```
//activity
 protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

}
//fragment
public static class ExampleFragment extends Fragment {
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        return inflater.inflate(R.layout.example_fragment, container, false);
    }
}
//view
 View.inflate(R.layout.example_fragment ,null, false);
//viewholder
public View getView(int position, View convertView, ViewGroup parent) {
            ViewHolder holder = new ViewHolder();
            if(convertView==null){
                convertView = inflater.inflate(R.layout.good_list_item, null, false);
                holder.img = (ImageView) convertView.findViewById(R.id.img);
                convertView.setTag(holder);
            }else{
                holder = (ViewHolder) convertView.getTag();
            }
            //设置holder
            holder.img.setImageResource(R.drawable.ic_launcher);


            return convertView;
}
//recycleView.viewholder
    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

        return new ItemViewHolder(LayoutInflater.from(mContext).inflate(R.layout.item_test, parent, false));
    }

    @Override
    public void onBindViewHolder(ViewHolder holder, int position) {
    }

    class ItemViewHolder extends ViewHolder {

        public ItemViewHolder(View itemView) {
            super(itemView);
        }
    }

```

以上不是一次友好的教学，也许你还在那样写，但是我拒绝。也许你可能看出了其中的问题，没错，代码风格不统一，接口不友好，view和逻辑没有分割违反了单一原则！
### 使用View-Layout
下面是采用View-Layout注入框架之后的规范代码：

```
//activity
@BindLayout(R.layout.activity_setting)
public class SettingActivity extends BaseTitleActivity {

}

/**
 * @author GanQuan
 * @since 2018/3/21.
 */
//fragment
@BindLayout(R.layout.activity_layout_main_home)
public class HomeTabFragment extends BaseTitleFragment {

}

//view
@BindLayout(R.layout.layout_main_item_header_login)
public class ReportProducedSucView extends BaseItemViewWithBean<HomeModule.HeaderBean> implements IHomeHeader {

}

//listview#viewholder or recycleView#viewholder(统一的风格)
  @BindLayout(R.layout.credit_detail_item_layout_overdue)
        static class ItemViewHolder extends BaseViewHolder<OverdueInfoBean.DetailsBean> {

}
```

所有view层接口代码都是通过依赖注入的方式去获取。
1.便于用户后期的查找xml；
2.代码更干净
3.弱化了视图和逻辑的交互，更好的提现的单一职责。

这一切都归功于View-Layout注入框架，它将视图层各种使用的场景处理和分发。统一了view层获取xml绑定的步骤，并且强制分割了view和model，自定义view可以跟卡片一样去管理。
### 原理和设计
    见github工程源码。
