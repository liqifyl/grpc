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

3.grpc_register_built_in_plugins方法定义在[grpc_plugin_registry.cc](https://github.com/grpc/grpc/blob/master/src/core/plugin_registry/grpc_plugin_registry.cc)文件中，本方法注册的相应插件请浏览源码

3. grpc_register_security_filters

4. register_builtin_channel_init方法定义在src/core/lib/surface/init.cc文件中

5. grpc_channel_init_finalize方法定义在src/core/lib/surface/channel_init.cc中

6. grpc_security_init方法定义在src/core/lib/surface/init_secure.cc

7. grpc_channel_init_register_stage方法定义在src/core/lib/surface/channel_init.cc中



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

4. grpc_channel_filter


# grpc connector 和 grpc subchannel

## grpc_connector结构体

1. grpc_connector结构体在src/core/ext/filters/client_channel/connector.h文件中
2. grpc_connector结构体中有类型grpc_connector_vtable的字段，grpc_connector_vtable结构体定义在src/core/ext/filters/client_channel/connector.h中

# rpc的创建

1.


