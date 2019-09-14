# grpc
grpc asyc stream

grpc asyc stream在释放时的顺序是stream->ctx->queue->channel->stub

# SSL
在使用grpc如果遇到{"tsi_code":10,"tsi_error":"TSI_PROTOCOL_FAILURE"}的错误，请检查客户端或者服务端当前时间

# grpc调试
export GRPC_VERBOSITY=DEBUG

# grpc args

1. grpc_arg结构体在grpc/impl/codegen/grpc_types.h文件中
2. grpc_channel_args结构体在grpc/impl/codegen/grpc_types.h文件中
