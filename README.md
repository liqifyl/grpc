# grpc
grpc asyc stream

grpc asyc stream在释放时的顺序是stream->ctx->queue->channel->stub

# SSL
在使用grpc如果遇到{"tsi_code":10,"tsi_error":"TSI_PROTOCOL_FAILURE"}的错误，请检查客户端或者服务端当前时间

# grpc调试
export GRPC_VERBOSITY=DEBUG

# grpc init流程

1. do_basic_init初始化基本资源，本方法内部会执行grpc_register_built_in_plugins方法注册所有插件，do_basic_init方法定义在src/core/lib/surface/init.cc文件中

2. grpc_init方法定义在src/core/lib/surface/init.cc文件中，本方法执行流程请浏览[init.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init.cc)

3. grpc_register_built_in_plugins方法定义在[grpc_plugin_registry.cc](https://github.com/grpc/grpc/blob/master/src/core/plugin_registry/grpc_plugin_registry.cc)文件中

4. grpc_register_security_filters

5. register_builtin_channel_init方法定义在src/core/lib/surface/init.cc文件中

6. grpc_channel_init_finalize方法定义在src/core/lib/surface/channel_init.cc中

7. grpc_security_init方法定义在src/core/lib/surface/init_secure.cc

8. grpc_channel_init_register_stage方法定义在src/core/lib/surface/channel_init.cc中

## grpc init流程中插件注册

### grpc_client_channel插件注册

* 调用grpc_register_plugin(grpc_client_channel_init,grpc_client_channel_shutdown)注册一个grpc_client_channel全局插件，grpc_register_plugin方法定义在[init.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init.cc)中，注册插件的方法会在grpc_init中调用

* grpc_client_channel_init方法定义在[client_channel_plugin.cc](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_plugin.cc)中，在grpc_client_channel_init方法中会调用grpc_channel_init_register_stage注册一个全局stage_slot,grpc_channel_init_register_stage方法定义在[channel_init.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)，这里grpc_channel_init_register_stage函数会传入2个参数，一个参数是append_filter函数，grpc_client_channel_filter

* append_filter何时被调用? 在调用[grpc_channel_create](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.cc)创建grpc_channel时在构建完[grpc_channel_stack_builder](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.h)后会调用[grpc_channel_init_create_stack](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)方法执行所有注册全局[stage_slot](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc),stage_slot结构体中定义一个fn字段用于执行注册的回调函数,从而也会调用appen_filter函数,append_filter函数会执行[grpc_channel_stack_builder_append_filter](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.cc)将[grpc_client_channel_filter](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)添加到channel_stack_builder结构体filter链表中

* 注册到channel_stack_builder结构体的grpc_cilent_channel_filter作用？grpc_client_channel_filter的类型是结构体[grpc_channel_filter](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/http/client/http_client_filter.cc);在创建grpc_channel时会触发[grpc_channel_create_with_builder](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.cc)，在grpc_channel_create_with_builder方法中会执行[grpc_channel_stack_builder_finish](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.cc)
方法，最终执行[grpc_channel_stack_init](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack.cc)方法，最终执行grpc_client_channel_filter变量中定义的init_channel_ele方法，grpc_client_channel_filter.init_channel_ele指针指向[ChannelData::Init](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc),ChannelData::Init方法中会创建一个ChannelData对象,在ChannelData构造函数中会调用[ClientChannelFactory::GetFromChannelArgs](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.cc)函数返回在[grpc_secure_channel_create](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)或者[grpc_insecure_channel_create](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)注册的[ClientChannelFactory](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.h)对象

* [ClientChannelFactory::CreateSubChannel](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.h)方法如何被调用？暂时猜测是在创建grpc_call时被创建，具体代码位置在[CallData::PickSubchannelLocked](https://github.com/grpc/grpc/blob/master/src/core/lib/transport/connectivity_state.cc)

# grpc args

1. grpc_arg结构体在grpc/impl/codegen/grpc_types.h文件中

2. grpc_channel_args结构体在grpc/impl/codegen/grpc_types.h文件中

3. grpc_channel_credentials结构体在src/core/lib/security/credentials/credentials.h文件中

4. grpc_channel_credentials_to_arg方法所在文件src/core/lib/security/credentials/credentials.cc

5. grpc_channel_arg_pointer_create创建指针类型的grpc_arg,grpc_channel_arg_pointer_create函数定义在src/core/lib/channel/channel_args.cc文件中

# grpc_impl::Channel创建流程

下面以创建SecureChannel为例：
grpc_channel结构体创建流程:grpc_impl::CreateCustomChannelImpl->grpc::ChannelCredentials(抽象类)::CreateChannelImpl->grpc::SecureChannelCredentials::CreateChannelImpl->grpc::SecureChannelCredentials::CreateChannelWithInterceptors->grpc_core::grpc_secure_channel_create->grpc_core::CreateChannel

1. grpc_impl::Channel类在src/cpp/client/channel_cc.cc文件中

2. grpc_impl::CreateCustomChannelImpl方法在src/cpp/client/create_channel.cc文件中

3. grpc::SecureChannelCredentials类在src/cpp/client/secure_credentials.cc文件中

4. grpc::SecureChannelCredentials::CreateChannelWithInterceptors方法中会调用grpc_core::grpc_secure_channel_create方法，grpc_core::grpc_secure_channel_create方法在src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc

5. grpc_core::CreateChannel方法定义在src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc中

## grpc_core::grpc_secure_channel_create注意事项

1. 本方法中会创建2个指针类型的grpc_arg参数，一个指针指向Chttp2SecureClientChannelFactory，一个指针指向grpc_channel_credentials

2. grpc_core::Chttp2SecureClientChannelFactory::CreateSubchannel函数何时被调用

## grpc_channel创建

1. grpc_channel_create->grpc_channel_create_with_builder创建grpc_channel，grpc_channel结构体定义在src/core/lib/surface/channel.h文件中

2. grpc_channel_init_create_stack方法定义在src/core/lib/surface/channel_init.cc中

3. grpc_channel_element和grpc_channel_filter结构体定义在src/core/lib/channel/channel_stack.h文件中

### grpc_channel_filter结构体分析




# grpc connector 和 grpc subchannel

## grpc_connector结构体

1. grpc_connector结构体在src/core/ext/filters/client_channel/connector.h文件中
2. grpc_connector结构体中有类型grpc_connector_vtable的字段，grpc_connector_vtable结构体定义在src/core/ext/filters/client_channel/connector.h中

# rpc的创建


