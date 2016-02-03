title: 关于view中measure方法测量出现偏差的bug
date: 2016-01-28 18:31:47
category: 问题记录
tags: [android,view,measure,布局错误]
---
项目中需要测量当前adapter的高度是否大于屏幕高度，代码如下：
```java
int itemViewType = mAdapter.getItemViewType(i);
if (itemViewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER && i > headerCount)
{
	//footerView高度不计算在内
	continue;
}
View view = mAdapter.getView(i, itemViewType > -1 && itemViewType < caches.length ? caches[itemViewType] : null, lv);
if (itemViewType > -1 && itemViewType < caches.length && caches[itemViewType] == null)
{
	caches[itemViewType] = view;
}
view.measure(widthMeasureSpec, heightMeasureSpec);
totalHeight += view.getMeasuredHeight() + dividerHeight;
if (totalHeight >= deviceHeight)
{
	oldIsLagerThanScreen = true;
	break;
}
```
即获取每一个item的高度相加。其中要测量的item布局如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                xmlns:app="http://schemas.android.com/apk/res/com.m4399.forums"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:background="@drawable/m4399_selector_adapter_item_bg"
                android:paddingTop="17dp"
                android:paddingBottom="17dp">

    <!-- 头像 -->
    <com.makeramen.RoundedImageView
        android:id="@+id/m4399_activity_common_user_info_list_item_header_icon_imv"
        android:layout_centerVertical="true"
        android:layout_width="48dip"
        android:layout_height="48dip"
        android:layout_gravity="center"
        android:scaleType="centerCrop"
        app:riv_corner_radius="24dip"
        android:layout_marginLeft="12dp"
        />

    <!-- 昵称和签名 -->
    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:layout_centerVertical="true"
        android:layout_marginLeft="12dp"
        android:layout_toRightOf="@+id/m4399_activity_common_user_info_list_item_header_icon_imv"
        android:layout_toLeftOf="@+id/m4399_activity_common_user_info_list_item_right_rl">
        <!--昵称-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_nick_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="17sp"
            android:textColor="@color/m4399_hei_333333"
            android:text="@string/m4399_feed_nick"/>
        <!--签名-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_sign_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="12sp"
            android:singleLine="true"
            android:textColor="@color/m4399_hui_999999"
            android:text="@string/m4399_feed_sign"/>
    </LinearLayout>

    <RelativeLayout
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_alignParentRight="true"
        android:paddingLeft="15dp"
        android:paddingRight="15dp"
        android:layout_centerInParent="true"
        android:id="@+id/m4399_activity_common_user_info_list_item_right_rl"
        >
        <!--关注-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_follow_tv"
            android:visibility="invisible"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:drawableTop="@drawable/m4399_ic_follow"
            android:layout_centerInParent="true"
            android:drawablePadding="5dp"
            android:textSize="14sp"
            android:textColor="@color/m4399_lan_4eb6f2"
            android:text="@string/m4399_common_follow"/>

        <!--已关注-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_followed_tv"
            android:visibility="gone"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:drawableTop="@drawable/m4399_ic_followed"
            android:layout_centerInParent="true"
            android:textColor="@color/m4399_hui_bbbbbb"
            android:drawablePadding="5dp"
            android:textSize="14sp"
            android:text="@string/m4399_common_followed"/>

        <!--互相关注-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_follow_each_other_tv"
            android:visibility="invisible"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:drawableTop="@drawable/m4399_ic_follow_mutual"
            android:layout_centerInParent="true"
            android:drawablePadding="5dp"
            android:textColor="@color/m4399_hui_bbbbbb"
            android:textSize="14sp"
            android:text="@string/m4399_feed_follow_each_other"/>


        <!-- 取消关注 -->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_follow_remove_tv"
            android:layout_width="@dimen/m4399_my_collection_quit_btn_width"
            android:layout_height="@dimen/m4399_my_collection_quit_btn_hight"
            android:background="@drawable/m4399_selector_btn_bg_red"
            android:visibility="gone"
            android:layout_centerInParent="true"
            android:gravity="center"
            android:clickable="true"
            android:text="@string/m4399_feed_follow_remove"
            android:textColor="@color/m4399_bai_FFFFFFFF"
            android:textSize="@dimen/m4399_my_collection_quit_btn_text_size" />
    </RelativeLayout>

</RelativeLayout>
```
其中中间的linearlayout设置了两个属性，leftof和rightof，这样布局并没有出现什么错误，也能得到想要的效果。但是用measure方法测量的结果却比实际的高度要多2倍之多。而且这个bug是在5.0以上的机型出现，5.0下测量值都是正常。bug出现后百思不得原因，后来觉得应该是布局出现了原因。于是定位到中间的linearlayout的两个冲突属性，修改布局文件如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
                xmlns:app="http://schemas.android.com/apk/res/com.m4399.forums"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:background="@drawable/m4399_selector_adapter_item_bg"
                android:paddingBottom="17dp"
                android:paddingTop="17dp">

    <!-- 头像 -->
    <com.makeramen.RoundedImageView
        android:id="@+id/m4399_activity_common_user_info_list_item_header_icon_imv"
        android:layout_width="48dip"
        android:layout_height="48dip"
        android:layout_centerVertical="true"
        android:layout_gravity="center"
        android:layout_marginLeft="12dp"
        android:scaleType="centerCrop"
        app:riv_corner_radius="24dip"
        />

    <!-- 昵称和签名 -->
    <LinearLayout
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="wrap_content"
        android:layout_gravity="center_vertical"
        android:layout_marginLeft="12dp"
        android:orientation="vertical">
        <!--昵称-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_nick_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/m4399_feed_nick"
            android:textColor="@color/m4399_hei_333333"
            android:textSize="17sp"/>
        <!--签名-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_sign_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:singleLine="true"
            android:text="@string/m4399_feed_sign"
            android:textColor="@color/m4399_hui_999999"
            android:textSize="12sp"/>
    </LinearLayout>

    <RelativeLayout
        android:id="@+id/m4399_activity_common_user_info_list_item_right_rl"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center_vertical"
        android:paddingLeft="15dp"
        android:paddingRight="15dp"
        >
        <!--关注-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_follow_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:drawablePadding="5dp"
            android:drawableTop="@drawable/m4399_ic_follow"
            android:text="@string/m4399_common_follow"
            android:textColor="@color/m4399_lan_4eb6f2"
            android:textSize="14sp"
            android:visibility="invisible"/>

        <!--已关注-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_followed_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:drawablePadding="5dp"
            android:drawableTop="@drawable/m4399_ic_followed"
            android:text="@string/m4399_common_followed"
            android:textColor="@color/m4399_hui_bbbbbb"
            android:textSize="14sp"
            android:visibility="gone"/>

        <!--互相关注-->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_follow_each_other_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:drawablePadding="5dp"
            android:drawableTop="@drawable/m4399_ic_follow_mutual"
            android:text="@string/m4399_feed_follow_each_other"
            android:textColor="@color/m4399_hui_bbbbbb"
            android:textSize="14sp"
            android:visibility="invisible"/>


        <!-- 取消关注 -->
        <TextView
            android:id="@+id/m4399_activity_common_user_info_list_item_follow_remove_tv"
            android:layout_width="@dimen/m4399_my_collection_quit_btn_width"
            android:layout_height="@dimen/m4399_my_collection_quit_btn_hight"
            android:layout_centerInParent="true"
            android:background="@drawable/m4399_selector_btn_bg_red"
            android:clickable="true"
            android:gravity="center"
            android:text="@string/m4399_feed_follow_remove"
            android:textColor="@color/m4399_bai_FFFFFFFF"
            android:textSize="@dimen/m4399_my_collection_quit_btn_text_size"
            android:visibility="gone"/>
    </RelativeLayout>

</LinearLayout>
```
利用linearlaout避免了使用两个冲突的属性，问题解决。
在代码中一定要使用规范的方式，而不是用看起来可以但是却略显蹩脚的方式实现需求。