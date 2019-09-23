# grpc
grpc asyc stream

grpc asyc stream在释放时的顺序是stream->ctx->queue->channel->stub

# SSL
在使用grpc如果遇到{"tsi_code":10,"tsi_error":"TSI_PROTOCOL_FAILURE"}的错误，请检查客户端或者服务端当前时间

# grpc调试
export GRPC_VERBOSITY=DEBUG

# grpc_impl::Channel对象创建流程

下面以创建SecureChannel为例：

[grpc_impl::CreateCustomChannelImpl](https://github.com/grpc/grpc/blob/master/src/cpp/client/create_channel.cc)->[grpc::ChannelCredentials(抽象类)::CreateChannelImpl](https://github.com/grpc/grpc/blob/master/include/grpcpp/security/credentials_impl.h)->[grpc::SecureChannelCredentials::CreateChannelImpl](https://github.com/grpc/grpc/blob/master/src/cpp/client/secure_credentials.cc)->[grpc::SecureChannelCredentials::CreateChannelWithInterceptors](https://github.com/grpc/grpc/blob/master/src/cpp/client/secure_credentials.cc)->[grpc_core::grpc_secure_channel_create](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)->[grpc_core::CreateChannel](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)->[grpc::CreateChannelInternal](https://github.com/grpc/grpc/blob/master/src/cpp/client/create_channel_internal.cc)

1. grpc_impl::Channel类在src/cpp/client/channel_cc.cc文件中

2. grpc_impl::CreateCustomChannelImpl方法在src/cpp/client/create_channel.cc文件中

3. grpc::SecureChannelCredentials类在src/cpp/client/secure_credentials.cc文件中

4. grpc::SecureChannelCredentials::CreateChannelWithInterceptors中会调用grpc_core::grpc_secure_channel_create方法，grpc_core::grpc_secure_channel_create方法定义在src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc，grpc_core::grpc_secure_channel_create中会调用grpc_core::CreateChannel方法，grpc_core::CreateChannel方法定义在src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc，grpc_core::CreateChannel方法返回[grpc_channel](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.h)结构体对象


## grpc_channel创建流程

* grpc_channel_create->grpc_channel_create_with_builder创建grpc_channel，grpc_channel结构体定义在src/core/lib/surface/channel.h文件中

* grpc_channel_init_create_stack方法定义在src/core/lib/surface/channel_init.cc中

* grpc_channel_element和grpc_channel_filter结构体定义在src/core/lib/channel/channel_stack.h文件中

## grpc::ChannelCredentials类注意事项

* static ::grpc::internal::GrpcLibraryInitializer g_gli_initialize全局变量定义在文件src/cpp/client/channel_cc.cc中，grpc::internal::GrpcLibraryInitializer类的构造函数会为grpc::g_glip实例化grpc::internal::GrpcLibrary对象和为g_core_codegen实例化grpc::internal::CoreCodegen对象

* grpc::internal::GrpcLibraryInitializer和grpc::internal::GrpcLibrary类定义在[grpc_library.h](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/grpc_library.h)文件中；grpc::internal::CoreCodegen类定义在[core_codegen.cc](https://github.com/grpc/grpc/blob/master/src/cpp/common/core_codegen.cc)中

* grpc::ChannelCredentials类继承自grpc::GrpcLibraryCodegen类,grpc::GrpcLibraryCodegen类构造函数会执行grpc::g_glip::init函数，grpc::g_glip::init函数会调用grpc::grpc_init


## grpc_init函数执行流程

[src/core/lib/surface/init.cc/grpc_init](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init.cc)函数会初始化所以需要的资源，下面分析几个重要函数：

* [do_basic_init](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init.cc)函数中会执行[grpc_register_built_in_plugins](https://github.com/grpc/grpc/blob/master/src/core/plugin_registry/grpc_plugin_registry.cc)方法注册所有插件

* [src/core/plugin_registry/grpc_plugin_registry.cc/grpc_register_built_in_plugins](https://github.com/grpc/grpc/blob/master/src/core/plugin_registry/grpc_plugin_registry.cc)函数会注册所有需要插件，包括channel、dns resolver、lb、http等，后面会详细分析

* [src/core/lib/surface/init.cc/register_builtin_channel_init](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init.cc)函数会注册一些默认插件，相应插件的含义不了解

* [src/core/lib/surface/channel_init.cc/grpc_channel_init_register_stage](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)函数用于保存注册插件信息
* [src/core/lib/surface/channel_init.cc/grpc_channel_init_finalize](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)函数用于销毁注册插件信息

* [src/core/lib/surface/init_secure.cc/grpc_security_init](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init_secure.cc)函数的含义暂时不了解

* [src/core/lib/surface/channel_init.cc/grpc_channel_init_create_stack](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)函数执行所有注册插件的回调函数，grpc_channel_init_create_stack在[src/core/lib/channel/channel_stack_builder.cc/grpc_channel_stack_builder_finish](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.cc)函数中被调用

从grpc_init函数主要作用是初始化所有需要的资源和注册相应需要的插件，注册的插件会在创建grpc_channel对象时调用后面会详解介绍；grpc_init函数使用到了全局变量g_initializations，grpc_init每初始化一次g_initializations加1，grpc_shutdown每调用一次g_initializations减1，g_initializations值变为0才会执行释放全局资源，如果存在多个grpc::channel只有所有grpc::channel销毁才会触发销毁全局资源;grpc为了防止内存泄漏导致释放全局资源是一个特别耗时操作，如果在grpc释放全局资源时同时创建channel可能存在阻塞一段时间，阻塞具体原因请查看[src/core/lib/iomgr/iomgr.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/iomgr/iomgr.cc)中grpc_iomgr_shutdown函数


# 插件注册流程和如何被执行

## grpc_client_channel插件注册

* 调用grpc_register_plugin(grpc_client_channel_init,grpc_client_channel_shutdown)注册一个grpc_client_channel全局插件，grpc_register_plugin方法定义在[init.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/init.cc)中，注册插件的方法会在grpc_init中调用

* grpc_client_channel_init方法定义在[client_channel_plugin.cc](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_plugin.cc)中，在grpc_client_channel_init方法中会调用grpc_channel_init_register_stage注册一个全局stage_slot,grpc_channel_init_register_stage方法定义在[channel_init.cc](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)，这里grpc_channel_init_register_stage函数会传入2个参数，一个参数是append_filter函数，grpc_client_channel_filter

* append_filter何时被调用? 在调用[grpc_channel_create](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.cc)创建grpc_channel时在构建完[grpc_channel_stack_builder](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.h)后会调用[grpc_channel_init_create_stack](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc)方法执行所有注册全局[stage_slot](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel_init.cc),stage_slot结构体中定义一个fn字段用于执行注册的回调函数,从而也会调用appen_filter函数,append_filter函数会执行[grpc_channel_stack_builder_append_filter](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.cc)将[grpc_client_channel_filter](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)添加到channel_stack_builder结构体filter链表中

* 注册到channel_stack_builder结构体的grpc_cilent_channel_filter作用？grpc_client_channel_filter的类型是结构体[grpc_channel_filter](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack.h);在创建grpc_channel时会触发[grpc_channel_create_with_builder](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.cc)，在grpc_channel_create_with_builder方法中会执行[grpc_channel_stack_builder_finish](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack_builder.cc)
方法，最终执行[grpc_channel_stack_init](https://github.com/grpc/grpc/blob/master/src/core/lib/channel/channel_stack.cc)方法，最终执行grpc_client_channel_filter变量中定义的init_channel_ele方法，grpc_client_channel_filter.init_channel_ele指针指向[ChannelData::Init](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc),ChannelData::Init方法中会创建一个ChannelData对象,在ChannelData构造函数中会调用[ClientChannelFactory::GetFromChannelArgs](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.cc)函数返回在[grpc_secure_channel_create](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)或者[grpc_insecure_channel_create](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)注册的[ClientChannelFactory](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.h)对象

* [ClientChannelFactory::CreateSubChannel](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.h)方法如何被调用？暂时猜测是在创建grpc_call时被创建，具体代码位置在[CallData::PickSubchannelLocked](https://github.com/grpc/grpc/blob/master/src/core/lib/transport/connectivity_state.cc)

# grpc::internal::BlockingUnaryCall创建过程

grpc::internal::BlockingUnaryCall是阻塞类型的call，创建流程如下：
* [grpc::internal::BlockingUnaryCall](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/client_unary_call.h)->[grpc_impl::Channel::CreateCall](https://github.com/grpc/grpc/blob/master/src/cpp/client/channel_cc.cc)->[grpc_impl::Channel::CreateCallInternal](https://github.com/grpc/grpc/blob/master/src/cpp/client/channel_cc.cc)->[grpc_channel_create_call](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.cc)->[grpc_channel_create_call_internal](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/channel.cc)->[grpc_call_create](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/call.cc);grpc_call_create内部返回[grpc_call](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/call.cc)对象;[grpc_impl::Channel::CreateCall](https://github.com/grpc/grpc/blob/master/src/cpp/client/channel_cc.cc)内部返回[grpc::internal::Call](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/call.h)

## grpc::internal::Call执行流程

* grpc::internal::Call的构造函数会传入3个参数分别是grpc_call、grpc::internal::CallHook、CompletionQueue，其中[grpc::internal::CallHook](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/call_hook.h)是个抽象类，从grpc_impl::Channel::CreateCallInternal函数内部看grpc::internal::CallHook指向grpc_impl::Channel,grpc::channel继承自grpc::internal::CallHook

* grpc::internal::BlockingUnaryCall->[grpc::internal::Call::PerformOps](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/call.h)->[grpc_impl:Channel::PerformOpsOnCall](https://github.com/grpc/grpc/blob/master/src/cpp/client/channel_cc.cc)->[grpc::internal::CallOpSetInterface::FillOps](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/call_op_set_interface.h)->[grpc::internal::CallOpSet::FillOps](https://github.com/grpc/grpc/blob/master/include/grpcpp/impl/codegen/call_op_set.h)->[grpc::CoreCodegen::grpc_call_start_batch](https://github.com/grpc/grpc/blob/master/src/cpp/common/core_codegen.cc)->[grpc::grpc_call_start_batch](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/call.cc)->[grpc::call_start_batch](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/call.cc)->[grpc::execute_batch](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/call.cc)->[grpc::execute_batch_in_call_combiner](https://github.com/grpc/grpc/blob/master/src/core/lib/surface/call.cc)->[grpc_channel_filter::start_transport_stream_op_batch]->[client_channel.cc::grpc_client_channel_filter::client_channel.cc](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[CallData::StartTransportStreamOpBatch](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[CallData::PickSubchannel](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[CallData::PickSubchannelLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[ChannelData::CheckConnectivityState](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[ChannelData::TryToConnectLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[ChannelData::CreateResolvingLoadBalancingPolicyLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[grpc_core::ResolvingLoadBalancingPolicy::ResolvingLoadBalancingPolicy](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/resolving_lb_policy.cc)->[grpc_core::AresDnsResolver::StartLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/resolver/dns/c_ares/dns_resolver_ares.cc)->[grpc_core::ResolverResultHandler::ReturnResult](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/resolving_lb_policy.cc)->[grpc_core::ResolvingLoadBalancingPolicy::OnResolverResultChangedLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/resolving_lb_policy.cc)->[grpc_core::ResolvingLoadBalancingPolicy::CreateOrUpdateLbPolicyLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/resolving_lb_policy.cc)->[grpc_core::ResolvingLoadBalancingPolicy::CreateLbPolicyLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/resolving_lb_policy.cc)->[grpc_core::RoundRobin::UpdateLocked](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/lb_policy/round_robin/round_robin.cc)->[grpc_core::SubchannelList::SubchannelList](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/lb_policy/subchannel_list.h)->[grpc_core::LoadBalancingPolicy::ChannelControlHelper::CreateSubchannel](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[grpc_core::ChannelData::Client_channel_factory::CreateSubchannel](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel.cc)->[grpc_core::ClientChannelFactory::CreateSubchannel](https://github.com/grpc/grpc/blob/master/src/core/ext/filters/client_channel/client_channel_factory.h)->[grpc_core::Chttp2SecureClientChannelFactory::CreateSubchannel](https://github.com/grpc/grpc/blob/master/src/core/ext/transport/chttp2/client/secure/secure_channel_create.cc)

# grpc args

1. grpc_arg结构体在grpc/impl/codegen/grpc_types.h文件中

2. grpc_channel_args结构体在grpc/impl/codegen/grpc_types.h文件中

3. grpc_channel_credentials结构体在src/core/lib/security/credentials/credentials.h文件中

4. grpc_channel_credentials_to_arg方法所在文件src/core/lib/security/credentials/credentials.cc

5. grpc_channel_arg_pointer_create创建指针类型的grpc_arg,grpc_channel_arg_pointer_create函数定义在src/core/lib/channel/channel_args.cc文件中



## grpc_core::grpc_secure_channel_create注意事项

1. 本方法中会创建2个指针类型的grpc_arg参数，一个指针指向Chttp2SecureClientChannelFactory，一个指针指向grpc_channel_credentials

2. grpc_core::Chttp2SecureClientChannelFactory::CreateSubchannel函数何时被调用

### grpc_channel_filter结构体分析




# grpc connector 和 grpc subchannel

## grpc_connector结构体

1. grpc_connector结构体在src/core/ext/filters/client_channel/connector.h文件中
2. grpc_connector结构体中有类型grpc_connector_vtable的字段，grpc_connector_vtable结构体定义在src/core/ext/filters/client_channel/connector.h中

# rpc的创建


