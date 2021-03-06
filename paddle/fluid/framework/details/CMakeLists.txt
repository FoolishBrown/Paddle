cc_library(var_handle SRCS var_handle.cc DEPS place framework_proto node)
cc_library(op_handle_base SRCS op_handle_base.cc DEPS var_handle device_context lod_tensor)
cc_library(op_graph_view SRCS op_graph_view.cc DEPS op_handle_base)
cc_library(scale_loss_grad_op_handle SRCS scale_loss_grad_op_handle.cc DEPS op_handle_base scope lod_tensor ddim memory)
cc_library(fetch_op_handle SRCS fetch_op_handle.cc DEPS op_handle_base scope lod_tensor ddim memory)
cc_library(computation_op_handle SRCS computation_op_handle.cc DEPS framework_proto scope place operator op_registry)
cc_library(rpc_op_handle SRCS rpc_op_handle.cc DEPS framework_proto scope place operator op_registry)

cc_library(multi_devices_helper SRCS multi_devices_helper.cc DEPS graph graph_helper)
cc_library(multi_devices_graph_print_pass SRCS multi_devices_graph_print_pass.cc DEPS multi_devices_helper)
cc_library(multi_devices_graph_check_pass SRCS multi_devices_graph_check_pass.cc DEPS multi_devices_helper)

cc_library(variable_visitor SRCS variable_visitor.cc DEPS lod_tensor selected_rows)

if(WITH_GPU)
    nv_library(all_reduce_op_handle SRCS all_reduce_op_handle.cc DEPS op_handle_base scope lod_tensor ddim memory
            dynload_cuda variable_visitor)
    if(WITH_DISTRIBUTE)
        nv_library(reduce_op_handle SRCS reduce_op_handle.cc DEPS op_handle_base variable_visitor scope
            ddim dynload_cuda selected_rows_functor sendrecvop_grpc)
    else()
        nv_library(reduce_op_handle SRCS reduce_op_handle.cc DEPS op_handle_base variable_visitor scope
            ddim dynload_cuda selected_rows_functor)
    endif()
    nv_library(broadcast_op_handle SRCS broadcast_op_handle.cc DEPS op_handle_base scope ddim memory variable_visitor dynload_cuda)
    nv_library(fused_broadcast_op_handle SRCS fused_broadcast_op_handle.cc DEPS broadcast_op_handle)

else()
    cc_library(all_reduce_op_handle SRCS all_reduce_op_handle.cc DEPS op_handle_base scope lod_tensor ddim memory
             variable_visitor)
    if(WITH_DISTRIBUTE)
        cc_library(reduce_op_handle SRCS reduce_op_handle.cc DEPS op_handle_base variable_visitor scope
            ddim selected_rows_functor sendrecvop_grpc)
    else()
        cc_library(reduce_op_handle SRCS reduce_op_handle.cc DEPS op_handle_base variable_visitor scope
            ddim selected_rows_functor)
    endif()
    cc_library(broadcast_op_handle SRCS broadcast_op_handle.cc DEPS op_handle_base scope ddim memory variable_visitor)
    cc_library(fused_broadcast_op_handle SRCS fused_broadcast_op_handle.cc DEPS broadcast_op_handle)
endif()

cc_library(data_balance_op_handle SRCS data_balance_op_handle.cc DEPS op_handle_base scope lod_tensor)
cc_library(gather_op_handle SRCS gather_op_handle.cc DEPS op_handle_base scope ddim memory variable_visitor)
cc_library(fuse_vars_op_handle SRCS fuse_vars_op_handle.cc DEPS op_handle_base scope)

cc_library(modify_op_lock_and_record_event_pass SRCS modify_op_lock_and_record_event_pass.cc DEPS computation_op_handle op_graph_view multi_devices_helper)

cc_library(reference_count_pass_helper SRCS reference_count_pass_helper.cc DEPS garbage_collector computation_op_handle)
cc_library(eager_deletion_op_handle SRCS eager_deletion_op_handle.cc DEPS lod_tensor selected_rows reference_count_pass_helper)
cc_library(eager_deletion_pass SRCS eager_deletion_pass.cc DEPS computation_op_handle eager_deletion_op_handle graph graph_helper pass)
cc_library(reference_count_pass SRCS reference_count_pass.cc DEPS computation_op_handle graph graph_helper pass op_graph_view reference_count_pass_helper)

cc_library(sequential_execution_pass SRCS sequential_execution_pass.cc DEPS graph graph_helper pass)
cc_library(all_reduce_deps_pass SRCS all_reduce_deps_pass.cc DEPS graph graph_helper pass)

cc_library(multi_devices_graph_pass SRCS multi_devices_graph_pass.cc DEPS multi_devices_helper computation_op_handle
        scale_loss_grad_op_handle rpc_op_handle all_reduce_op_handle reduce_op_handle broadcast_op_handle data_balance_op_handle fused_broadcast_op_handle)

set(SSA_GRAPH_EXECUTOR_DEPS graph framework_proto sequential_execution_pass modify_op_lock_and_record_event_pass all_reduce_deps_pass reference_count_pass eager_deletion_pass) 

cc_library(ssa_graph_executor SRCS ssa_graph_executor.cc DEPS ${SSA_GRAPH_EXECUTOR_DEPS})

cc_library(threaded_ssa_graph_executor SRCS threaded_ssa_graph_executor.cc DEPS fetch_op_handle ssa_graph_executor scope
        simple_threadpool device_context)

cc_test(broadcast_op_test SRCS broadcast_op_handle_test.cc DEPS var_handle op_handle_base scope ddim memory
        device_context broadcast_op_handle)
cc_test(gather_op_test SRCS gather_op_handle_test.cc DEPS var_handle op_handle_base scope ddim memory
        device_context gather_op_handle)
cc_library(scope_buffered_ssa_graph_executor SRCS scope_buffered_ssa_graph_executor.cc DEPS ssa_graph_executor)
#cc_test(reduce_op_handle_test SRCS reduce_op_handle_test.cc DEPS var_handle op_handle_base scope ddim memory
#        device_context reduce_op_handle )
cc_library(fast_threaded_ssa_graph_executor SRCS fast_threaded_ssa_graph_executor.cc
        DEPS fetch_op_handle ssa_graph_executor scope simple_threadpool device_context)
cc_test(fused_broadcast_op_test SRCS fused_broadcast_op_handle_test.cc DEPS fused_broadcast_op_handle)

cc_library(build_strategy SRCS build_strategy.cc DEPS
        graph_viz_pass multi_devices_graph_pass
        multi_devices_graph_print_pass multi_devices_graph_check_pass
        fuse_elewise_add_act_pass multi_batch_merge_pass)
