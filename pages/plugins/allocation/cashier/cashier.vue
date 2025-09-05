<template>
    <view :class="theme_view">
        <view class="page bg-white">
            <view class="content padding-horizontal-main tc">
                <block v-if="data_list_loding_status == 3">
                    <view class="padding-vertical-xxxxl">
                        <text class="cr-price fw-b text-size-lg">{{payment_currency_symbol}}</text>
                        <text class="cr-price fw-b text-size-xxl">{{data.pay_price}}</text>
                    </view>
                    <view class="margin-top-sm padding-bottom-xxxxl">
                        <view :class="'cr-'+(pay_status == 1 ? 'green' : (pay_status == 2 ? 'red' : 'grey'))">{{pay_msg}}</view>
                    </view>
                    <view v-if="pay_status == 2" class="margin-top-xxxxl padding-vertical-xxxxl">
                        <button class="bg-green br-green cr-white round text-size-sm padding-horizontal-xxxxl" size="mini" hover-class="none" @tap="pay_handle">重新发起支付</button>
                    </view>
                </block>
                <block v-else>
                    <!-- 提示信息 -->
                    <component-no-data :propStatus="data_list_loding_status" :propMsg="data_list_loding_msg"></component-no-data>
                </block>
                
                <view v-if="is_back_btn && pay_status != 0" class="margin-top-xxxxl padding-top-xxxxl tc">
                    <button class="bg-white br-main cr-main round text-size-sm padding-horizontal-xxxxl" size="mini" hover-class="none" open-type="launchApp" app-parameter="wechat">返回APP</button>
                </view>
            </view>
        </view>

        <!-- 公共 -->
        <component-common ref="common" :propIsAppAdmin="false"></component-common>
    </view>
</template>
<script>
    const app = getApp();
    import componentCommon from '@/components/common/common';
    import componentNoData from "@/components/no-data/no-data";
    export default {
        data() {
            return {
                theme_view: app.globalData.get_theme_value_view(),
                payment_currency_symbol: app.globalData.currency_symbol(),
                params: {},
                data_list_loding_status: 1,
                data_list_loding_msg: '',
                scene: 0,
                scene_back_arr: [1069, 1038, 1089, 1090, 1001, 1131, 1187],
                is_back_btn: false,
                data: {},
                pay_status: 0,
                pay_msg: '',
            };
        },

        components: {
            componentCommon,
            componentNoData
        },
        onLoad(params) {
            // 调用公共事件方法
            app.globalData.page_event_onload_handle(params);

            // 设置参数
            params = app.globalData.launch_params_handle(params);
            var scene = app.globalData.get_scene_data();
            this.setData({
                params: params,
                scene: scene,
                is_back_btn: this.scene_back_arr.indexOf(scene) != -1,
            });

            // 数据加载
            this.get_data();
        },

        onShow() {
            // 调用公共事件方法
            app.globalData.page_event_onshow_handle();

            // 公共onshow事件
            if ((this.$refs.common || null) != null) {
                this.$refs.common.on_show();
            }
        },

        // 下拉刷新
        onPullDownRefresh() {
            this.get_data();
        },

        methods: {
            // // 初始化
            get_data() {
                var self = this;
                var action = 'login';
                // #ifdef MP-BAIDU
                action = 'getLoginCode';
                // #endif
                uni[action]({
                    success: (res) => {
                        if (res.code) {
                            uni.request({
                                url: app.globalData.get_request_url("paydata", "cashier", "allocation"),
                                method: 'POST',
                                data: {...self.params, ...{authcode: res.code}},
                                dataType: 'json',
                                success: (res) => {
                                    uni.stopPullDownRefresh();
                                    if (res.data.code == 0 && (res.data.data || null) != null) {
                                        this.setData({
                                            data_list_loding_status: 3,
                                            data_list_loding_msg: '',
                                            data: res.data.data,
                                        });
                                    
                                        // 直接调起支付
                                        this.pay_handle();
                                    } else {
                                        this.setData({
                                            data_list_loding_status: 0,
                                            data_list_loding_msg: res.data.msg,
                                        });
                                    }
                                },
                                fail: () => {
                                    uni.stopPullDownRefresh();
                                    this.setData({
                                        data_list_loding_status: 0,
                                        data_list_loding_msg: self.$t('common.internet_error_tips'),
                                    });
                                },
                            });
                        }
                    },
                    fail: (e) => {
                        uni.stopPullDownRefresh();
                        this.setData({
                            data_list_loding_status: 0,
                            data_list_loding_msg: self.$t('login.login.3nmrg2'),
                        });
                    },
                });
            },

            // 支付处理
            pay_handle(e) {
                if(this.pay_status != 1)
                {
                    this.setData({
                        pay_status: 0,
                        pay_msg: this.$t('common.payment_in_text'),
                    });
                    
                    // 向后兼容：收银台微信支付参数验证（兼容v2和v3）
                    if (!this.validateWeChatPayParams(this.data.pay_data)) {
                        this.setData({
                            pay_status: 2,
                            pay_msg: '微信支付参数验证失败',
                        });
                        return;
                    }
                    
                    // 向后兼容：构造标准支付参数（保持v2格式兼容性）
                    const paymentParams = {
                        timeStamp: this.data.pay_data.timeStamp,
                        nonceStr: this.data.pay_data.nonceStr,
                        package: this.data.pay_data.package,
                        signType: this.data.pay_data.signType,
                        paySign: this.data.pay_data.paySign
                    };
                    
                    // 如果有appId参数（v3格式），也包含进去
                    if (this.data.pay_data.appId) {
                        paymentParams.appId = this.data.pay_data.appId;
                    }
                    
                    // 兼容性日志
                    const isV3Format = this.data.pay_data.appId && this.data.pay_data.signType === 'RSA';
                    console.log(`收银台微信支付 (${isV3Format ? 'v3' : 'v2'}格式):`, paymentParams);
                    
                    // 微信支付调用（保持原有接口不变）
                    uni.requestPayment({
                        ...paymentParams,
                        success: (res) => {
                            console.log('收银台微信支付成功：', res);
                            this.setData({
                                pay_status: 1,
                                pay_msg: this.$t('paytips.paytips.679rxu'),
                            });
                        },
                        fail: (res) => {
                            console.error('收银台微信支付失败：', res);
                            this.setData({
                                pay_status: 2,
                                pay_msg: this.getPaymentErrorMessage(res),
                            });
                        },
                    });
                }
            },

            // 向后兼容：收银台参数验证方法（兼容v2和v3）
            validateWeChatPayParams(payData) {
                if (!payData) {
                    console.error('收银台微信支付：支付参数为空');
                    return false;
                }
                
                // 通用必要参数验证
                const requiredParams = ['timeStamp', 'nonceStr', 'package', 'signType', 'paySign'];
                for (let param of requiredParams) {
                    if (!payData[param] || payData[param].toString().trim() === '') {
                        console.error(`收银台微信支付：缺少必要参数 ${param}`);
                        return false;
                    }
                }
                
                // 检测并验证格式
                const isV3Format = payData.appId && payData.signType === 'RSA';
                if (isV3Format) {
                    // v3格式额外验证appId
                    if (!/^wx[a-zA-Z0-9]{16}$/.test(payData.appId)) {
                        console.error('收银台微信支付v3：appId格式错误');
                        return false;
                    }
                    console.log('收银台检测到微信支付v3格式参数');
                } else {
                    console.log('收银台检测到微信支付v2格式参数');
                }
                
                return true;
            },

            // 向后兼容：收银台错误消息处理
            getPaymentErrorMessage(res) {
                const errorCode = res.errCode || res.errNo || null;
                switch (errorCode) {
                    case -1:
                        return '用户取消支付';
                    case -2:
                        return '网络错误，请重试';
                    case -3:
                        return '系统错误，请稍后重试';
                    default:
                        return res.errMsg || this.$t('paytips.paytips.6y488i');
                }
            }
        }
    };
</script>
<style>
    @import './cashier.css';
</style>