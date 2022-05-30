# DirectX Raytracing (DXR) Functional Spec <!-- omit in toc -->

## v1.18 3/31/2022

# Contents <!-- omit in toc -->

- [Intro](#intro)
- [Overview](#overview)
- [Design goals](#design-goals)
- [Walkthrough](#walkthrough)
  - [Initiating raytracing](#initiating-raytracing)
  - [Ray generation shaders](#ray-generation-shaders)
  - [Rays](#rays)
  - [Raytracing output](#raytracing-output)
  - [Ray-geometry interaction diagram](#ray-geometry-interaction-diagram)
  - [Geometry and acceleration structures](#geometry-and-acceleration-structures)
  - [Acceleration structure updates](#acceleration-structure-updates)
  - [Built-in ray-triangle intersection - triangle mesh geometry](#built-in-ray-triangle-intersection---triangle-mesh-geometry)
  - [Intersection shaders - procedural primitive geometry](#intersection-shaders---procedural-primitive-geometry)
    - [Minor intersection shader details](#minor-intersection-shader-details)
  - [Any hit shaders](#any-hit-shaders)
  - [Closest hit shaders](#closest-hit-shaders)
  - [Miss shaders](#miss-shaders)
  - [Hit groups](#hit-groups)
  - [TraceRay control flow](#traceray-control-flow)
  - [Flags per ray](#flags-per-ray)
  - [Instance masking](#instance-masking)
  - [Callable shaders](#callable-shaders)
  - [Resource binding](#resource-binding)
    - [Local root signatures vs global root signatures](#local-root-signatures-vs-global-root-signatures)
  - [Shader identifier](#shader-identifier)
  - [Shader record](#shader-record)
  - [Shader tables](#shader-tables)
  - [Indexing into shader tables](#indexing-into-shader-tables)
    - [Shader record stride](#shader-record-stride)
    - [Shader table memory initialization](#shader-table-memory-initialization)
  - [Inline raytracing](#inline-raytracing)
    - [TraceRayInline control flow](#tracerayinline-control-flow)
    - [Specialized TraceRayInline control flow](#specialized-tracerayinline-control-flow)
- [Shader management](#shader-management)
  - [Problem space](#problem-space)
    - [Implementations juggle many shaders](#implementations-juggle-many-shaders)
    - [Applications control shader compilation](#applications-control-shader-compilation)
  - [State objects](#state-objects)
    - [Subobjects](#subobjects)
      - [Subobjects in DXIL libraries](#subobjects-in-dxil-libraries)
    - [State object types](#state-object-types)
      - [Raytracing pipeline state object](#raytracing-pipeline-state-object)
      - [Graphics and compute state objects](#graphics-and-compute-state-objects)
      - [Collection state object](#collection-state-object)
      - [Collections vs libraries](#collections-vs-libraries)
    - [DXIL libraries and state objects example](#dxil-libraries-and-state-objects-example)
    - [Subobject association behavior](#subobject-association-behavior)
      - [Default associations](#default-associations)
        - [Terminology](#terminology)
        - [Declaring a default association](#declaring-a-default-association)
        - [Behavior of a default association](#behavior-of-a-default-association)
      - [Explicit associations](#explicit-associations)
      - [Multiple associations of a subobject](#multiple-associations-of-a-subobject)
      - [Conflicting subobject associations](#conflicting-subobject-associations)
        - [Exception: overriding DXIL library associations](#exception-overriding-dxil-library-associations)
      - [Subobject associations for hit groups](#subobject-associations-for-hit-groups)
      - [Runtime resolves associations for driver](#runtime-resolves-associations-for-driver)
    - [Subobject association requirements](#subobject-association-requirements)
    - [State object caching](#state-object-caching)
    - [Incremental additions to existing state objects](#incremental-additions-to-existing-state-objects)
- [System limits and fixed function behaviors](#system-limits-and-fixed-function-behaviors)
  - [Addressing calculations within shader tables](#addressing-calculations-within-shader-tables)
    - [Hit group table indexing](#hit-group-table-indexing)
    - [Miss shader table indexing](#miss-shader-table-indexing)
    - [Callable shader table indexing](#callable-shader-table-indexing)
    - [Out of bounds shader table indexing](#out-of-bounds-shader-table-indexing)
  - [Acceleration structure properties](#acceleration-structure-properties)
    - [Data rules](#data-rules)
    - [Determinism based on fixed acceleration structure build input](#determinism-based-on-fixed-acceleration-structure-build-input)
    - [Determinism based varying acceleration structure build input](#determinism-based-varying-acceleration-structure-build-input)
    - [Preservation of triangle set](#preservation-of-triangle-set)
    - [AABB volume](#aabb-volume)
    - [Inactive primitives and instances](#inactive-primitives-and-instances)
    - [Degenerate primitives and instances](#degenerate-primitives-and-instances)
    - [Geometry limits](#geometry-limits)
  - [Acceleration structure update constraints](#acceleration-structure-update-constraints)
    - [Bottom-level acceleration structure updates](#bottom-level-acceleration-structure-updates)
    - [Top-level acceleration structure updates](#top-level-acceleration-structure-updates)
  - [Acceleration structure memory restrictions](#acceleration-structure-memory-restrictions)
    - [Synchronizing acceleration structure memory writes/reads](#synchronizing-acceleration-structure-memory-writesreads)
  - [Fixed function ray-triangle intersection specification](#fixed-function-ray-triangle-intersection-specification)
    - [Watertightness](#watertightness)
      - [Top-left rule](#top-left-rule)
    - [Example top-left rule implementation](#example-top-left-rule-implementation)
      - [Determining a coordinate system](#determining-a-coordinate-system)
      - [Hypothetical scheme for establishing plane for ray-tri intersection](#hypothetical-scheme-for-establishing-plane-for-ray-tri-intersection)
      - [Triangle intersection](#triangle-intersection)
      - [Examples of classifying triangle edges](#examples-of-classifying-triangle-edges)
  - [Ray extents](#ray-extents)
  - [Ray recursion limit](#ray-recursion-limit)
  - [Pipeline stack](#pipeline-stack)
    - [Optimal pipeline stack size calculation](#optimal-pipeline-stack-size-calculation)
    - [Default pipeline stack size](#default-pipeline-stack-size)
    - [Pipeline stack limit behavior](#pipeline-stack-limit-behavior)
  - [Shader limitations resulting from independence](#shader-limitations-resulting-from-independence)
    - [Wave Intrinsics](#wave-intrinsics)
  - [Execution and memory ordering](#execution-and-memory-ordering)
- [General tips for building acceleration structures](#general-tips-for-building-acceleration-structures)
  - [Choosing acceleration structure build flags](#choosing-acceleration-structure-build-flags)
- [Determining raytracing support](#determining-raytracing-support)
  - [Raytracing emulation](#raytracing-emulation)
- [Tools support](#tools-support)
  - [Buffer bounds tracking](#buffer-bounds-tracking)
  - [Acceleration structure processing](#acceleration-structure-processing)
- [API](#api)
  - [Device methods](#device-methods)
    - [CheckFeatureSupport](#checkfeaturesupport)
      - [CheckFeatureSupport Structures](#checkfeaturesupport-structures)
        - [D3D12_FEATURE_D3D12_OPTIONS5](#d3d12_feature_d3d12_options5)
        - [D3D12_RAYTRACING_TIER](#d3d12_raytracing_tier)
    - [CreateStateObject](#createstateobject)
      - [CreateStateObject Structures](#createstateobject-structures)
        - [D3D12_STATE_OBJECT_DESC](#d3d12_state_object_desc)
        - [D3D12_STATE_OBJECT_TYPE](#d3d12_state_object_type)
        - [D3D12_STATE_SUBOBJECT](#d3d12_state_subobject)
        - [D3D12_STATE_SUBOBJECT_TYPE](#d3d12_state_subobject_type)
        - [D3D12_STATE_OBJECT_CONFIG](#d3d12_state_object_config)
        - [D3D12_STATE_OBJECT_FLAGS](#d3d12_state_object_flags)
        - [D3D12_GLOBAL_ROOT_SIGNATURE](#d3d12_global_root_signature)
        - [D3D12_LOCAL_ROOT_SIGNATURE](#d3d12_local_root_signature)
        - [D3D12_DXIL_LIBRARY_DESC](#d3d12_dxil_library_desc)
        - [D3D12_EXPORT_DESC](#d3d12_export_desc)
        - [D3D12_EXPORT_FLAGS](#d3d12_export_flags)
        - [D3D12_EXISTING_COLLECTION_DESC](#d3d12_existing_collection_desc)
        - [D3D12_HIT_GROUP_DESC](#d3d12_hit_group_desc)
        - [D3D12_HIT_GROUP_TYPE](#d3d12_hit_group_type)
        - [D3D12_RAYTRACING_SHADER_CONFIG](#d3d12_raytracing_shader_config)
        - [D3D12_RAYTRACING_PIPELINE_CONFIG](#d3d12_raytracing_pipeline_config)
        - [D3D12_RAYTRACING_PIPELINE_CONFIG1](#d3d12_raytracing_pipeline_config1)
        - [D3D12_RAYTRACING_PIPELINE_FLAGS](#d3d12_raytracing_pipeline_flags)
        - [D3D12_NODE_MASK](#d3d12_node_mask)
        - [D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION](#d3d12_subobject_to_exports_association)
        - [D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION](#d3d12_dxil_subobject_to_exports_association)
    - [AddToStateObject](#addtostateobject)
    - [GetRaytracingAccelerationStructurePrebuildInfo](#getraytracingaccelerationstructureprebuildinfo)
      - [GetRaytracingAccelerationStructurePrebuildInfo Structures](#getraytracingaccelerationstructureprebuildinfo-structures)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO](#d3d12_raytracing_acceleration_structure_prebuild_info)
    - [CheckDriverMatchingIdentifier](#checkdrivermatchingidentifier)
      - [CheckDriverMatchingIdentifier Structures](#checkdrivermatchingidentifier-structures)
        - [D3D12_SERIALIZED_DATA_TYPE](#d3d12_serialized_data_type)
        - [D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER](#d3d12_serialized_data_driver_matching_identifier)
        - [D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS](#d3d12_driver_matching_identifier_status)
    - [CreateCommandSignature](#createcommandsignature)
      - [CreateCommandSignature Structures](#createcommandsignature-structures)
        - [D3D12_COMMAND_SIGNATURE_DESC](#d3d12_command_signature_desc)
        - [D3D12_INDIRECT_ARGUMENT_DESC](#d3d12_indirect_argument_desc)
        - [D3D12_INDIRECT_ARGUMENT_TYPE](#d3d12_indirect_argument_type)
  - [Command list methods](#command-list-methods)
    - [BuildRaytracingAccelerationStructure](#buildraytracingaccelerationstructure)
      - [BuildRaytracingAccelerationStructure Structures](#buildraytracingaccelerationstructure-structures)
        - [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC](#d3d12_build_raytracing_acceleration_structure_desc)
        - [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS](#d3d12_build_raytracing_acceleration_structure_inputs)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE](#d3d12_raytracing_acceleration_structure_type)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS](#d3d12_raytracing_acceleration_structure_build_flags)
        - [D3D12_ELEMENTS_LAYOUT](#d3d12_elements_layout)
        - [D3D12_RAYTRACING_GEOMETRY_DESC](#d3d12_raytracing_geometry_desc)
        - [D3D12_RAYTRACING_GEOMETRY_TYPE](#d3d12_raytracing_geometry_type)
        - [D3D12_RAYTRACING_GEOMETRY_FLAGS](#d3d12_raytracing_geometry_flags)
        - [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc)
        - [D3D12_RAYTRACING_GEOMETRY_AABBS_DESC](#d3d12_raytracing_geometry_aabbs_desc)
        - [D3D12_RAYTRACING_AABB](#d3d12_raytracing_aabb)
        - [D3D12_RAYTRACING_INSTANCE_DESC](#d3d12_raytracing_instance_desc)
        - [D3D12_RAYTRACING_INSTANCE_FLAGS](#d3d12_raytracing_instance_flags)
        - [D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE](#d3d12_gpu_virtual_address_and_stride)
    - [EmitRaytracingAccelerationStructurePostbuildInfo](#emitraytracingaccelerationstructurepostbuildinfo)
      - [EmitRaytracingAccelerationStructurePostbuildInfo Structures](#emitraytracingaccelerationstructurepostbuildinfo-structures)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_desc)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TYPE](#d3d12_raytracing_acceleration_structure_postbuild_info_type)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_compacted_size_desc)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_tools_visualization_desc)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_serialization_desc)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_current_size_desc)
    - [CopyRaytracingAccelerationStructure](#copyraytracingaccelerationstructure)
      - [CopyRaytracingAccelerationStructure Structures](#copyraytracingaccelerationstructure-structures)
        - [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE](#d3d12_raytracing_acceleration_structure_copy_mode)
        - [D3D12_SERIALIZED_ACCELERATION_STRUCTURE_HEADER](#d3d12_serialized_acceleration_structure_header)
        - [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_TOOLS_VISUALIZATION_HEADER](#d3d12_build_raytracing_acceleration_structure_tools_visualization_header)
    - [SetPipelineState1](#setpipelinestate1)
    - [DispatchRays](#dispatchrays)
      - [DispatchRays Structures](#dispatchrays-structures)
        - [D3D12_DISPATCH_RAYS_DESC](#d3d12_dispatch_rays_desc)
        - [D3D12_GPU_VIRTUAL_ADDRESS_RANGE](#d3d12_gpu_virtual_address_range)
        - [D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE](#d3d12_gpu_virtual_address_range_and_stride)
    - [ExecuteIndirect](#executeindirect)
  - [ID3D12StateObjectProperties methods](#id3d12stateobjectproperties-methods)
    - [GetShaderIdentifier](#getshaderidentifier)
    - [GetShaderStackSize](#getshaderstacksize)
    - [GetPipelineStackSize](#getpipelinestacksize)
    - [SetPipelineStackSize](#setpipelinestacksize)
  - [Additional resource states](#additional-resource-states)
  - [Additional root signature flags](#additional-root-signature-flags)
    - [D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE](#d3d12_root_signature_flag_local_root_signature)
    - [Note on shader visibility](#note-on-shader-visibility)
  - [Additional SRV type](#additional-srv-type)
  - [Constants](#constants)
- [HLSL](#hlsl)
  - [Types, enums, subobjects and concepts](#types-enums-subobjects-and-concepts)
    - [Ray flags](#ray-flags)
    - [Ray description structure](#ray-description-structure)
    - [Raytracing pipeline flags](#raytracing-pipeline-flags)
    - [RaytracingAccelerationStructure](#raytracingaccelerationstructure)
    - [Subobject definitions](#subobject-definitions)
      - [Hit group](#hit-group)
      - [Root signature](#root-signature)
      - [Local root signature](#local-root-signature)
      - [Subobject to entrypoint association](#subobject-to-entrypoint-association)
      - [Raytracing shader config](#raytracing-shader-config)
      - [Raytracing pipeline config](#raytracing-pipeline-config)
      - [Raytracing pipeline config1](#raytracing-pipeline-config1)
    - [Intersection attributes structure](#intersection-attributes-structure)
    - [Ray payload structure](#ray-payload-structure)
    - [Call parameter structure](#call-parameter-structure)
  - [Shaders](#shaders)
    - [Ray generation shader](#ray-generation-shader)
    - [Intersection shader](#intersection-shader)
    - [Any hit shader](#any-hit-shader)
    - [Closest hit shader](#closest-hit-shader)
    - [Miss shader](#miss-shader)
    - [Callable shader](#callable-shader)
  - [Intrinsics](#intrinsics)
    - [CallShader](#callshader)
    - [TraceRay](#traceray)
    - [ReportHit](#reporthit)
    - [IgnoreHit](#ignorehit)
    - [AcceptHitAndEndSearch](#accepthitandendsearch)
  - [System value intrinsics](#system-value-intrinsics)
    - [Ray dispatch system values](#ray-dispatch-system-values)
      - [DispatchRaysIndex](#dispatchraysindex)
      - [DispatchRaysDimensions](#dispatchraysdimensions)
    - [Ray system values](#ray-system-values)
      - [WorldRayOrigin](#worldrayorigin)
      - [WorldRayDirection](#worldraydirection)
      - [RayTMin](#raytmin)
      - [RayTCurrent](#raytcurrent)
      - [RayFlags](#rayflags)
    - [Primitive/object space system values](#primitiveobject-space-system-values)
      - [InstanceIndex](#instanceindex)
      - [InstanceID](#instanceid)
      - [GeometryIndex](#geometryindex)
      - [PrimitiveIndex](#primitiveindex)
      - [ObjectRayOrigin](#objectrayorigin)
      - [ObjectRayDirection](#objectraydirection)
      - [ObjectToWorld3x4](#objecttoworld3x4)
      - [ObjectToWorld4x3](#objecttoworld4x3)
      - [WorldToObject3x4](#worldtoobject3x4)
      - [WorldToObject4x3](#worldtoobject4x3)
    - [Hit specific system values](#hit-specific-system-values)
      - [HitKind](#hitkind)
  - [RayQuery](#rayquery)
    - [RayQuery intrinsics](#rayquery-intrinsics)
      - [RayQuery enums](#rayquery-enums)
        - [COMMITTED_STATUS](#committed_status)
        - [CANDIDATE_TYPE](#candidate_type)
      - [RayQuery TraceRayInline](#rayquery-tracerayinline)
        - [TraceRayInline examples](#tracerayinline-examples)
          - [TraceRayInline example 1](#tracerayinline-example-1)
          - [TraceRayInline example 2](#tracerayinline-example-2)
          - [TraceRayInline example 3](#tracerayinline-example-3)
      - [RayQuery Proceed](#rayquery-proceed)
      - [RayQuery Abort](#rayquery-abort)
      - [RayQuery CandidateType](#rayquery-candidatetype)
      - [RayQuery CandidateProceduralPrimitiveNonOpaque](#rayquery-candidateproceduralprimitivenonopaque)
      - [RayQuery CommitNonOpaqueTriangleHit](#rayquery-commitnonopaquetrianglehit)
      - [RayQuery CommitProceduralPrimitiveHit](#rayquery-commitproceduralprimitivehit)
      - [RayQuery CommittedStatus](#rayquery-committedstatus)
      - [RayQuery RayFlags](#rayquery-rayflags)
      - [RayQuery WorldRayOrigin](#rayquery-worldrayorigin)
      - [RayQuery WorldRayDirection](#rayquery-worldraydirection)
      - [RayQuery RayTMin](#rayquery-raytmin)
      - [RayQuery CandidateTriangleRayT](#rayquery-candidatetrianglerayt)
      - [RayQuery CommittedRayT](#rayquery-committedrayt)
      - [RayQuery CandidateInstanceIndex](#rayquery-candidateinstanceindex)
      - [RayQuery CandidateInstanceID](#rayquery-candidateinstanceid)
      - [RayQuery CandidateInstanceContributionToHitGroupIndex](#rayquery-candidateinstancecontributiontohitgroupindex)
      - [RayQuery CandidateGeometryIndex](#rayquery-candidategeometryindex)
      - [RayQuery CandidatePrimitiveIndex](#rayquery-candidateprimitiveindex)
      - [RayQuery CandidateObjectRayOrigin](#rayquery-candidateobjectrayorigin)
      - [RayQuery CandidateObjectRayDirection](#rayquery-candidateobjectraydirection)
      - [RayQuery CandidateObjectToWorld3x4](#rayquery-candidateobjecttoworld3x4)
      - [RayQuery CandidateObjectToWorld4x3](#rayquery-candidateobjecttoworld4x3)
      - [RayQuery CandidateWorldToObject3x4](#rayquery-candidateworldtoobject3x4)
      - [RayQuery CandidateWorldToObject4x3](#rayquery-candidateworldtoobject4x3)
      - [RayQuery CommittedInstanceIndex](#rayquery-committedinstanceindex)
      - [RayQuery CommittedInstanceID](#rayquery-committedinstanceid)
      - [RayQuery CommittedInstanceContributionToHitGroupIndex](#rayquery-committedinstancecontributiontohitgroupindex)
      - [RayQuery CommittedGeometryIndex](#rayquery-committedgeometryindex)
      - [RayQuery CommittedPrimitiveIndex](#rayquery-committedprimitiveindex)
      - [RayQuery CommittedObjectRayOrigin](#rayquery-committedobjectrayorigin)
      - [RayQuery CommittedObjectRayDirection](#rayquery-committedobjectraydirection)
      - [RayQuery CommittedObjectToWorld3x4](#rayquery-committedobjecttoworld3x4)
      - [RayQuery CommittedObjectToWorld4x3](#rayquery-committedobjecttoworld4x3)
      - [RayQuery CommittedWorldToObject3x4](#rayquery-committedworldtoobject3x4)
      - [RayQuery CommittedWorldToObject4x3](#rayquery-committedworldtoobject4x3)
      - [RayQuery CandidateTriangleBarycentrics](#rayquery-candidatetrianglebarycentrics)
      - [RayQuery CandidateTriangleFrontFace](#rayquery-candidatetrianglefrontface)
      - [RayQuery CommittedTriangleBarycentrics](#rayquery-committedtrianglebarycentrics)
      - [RayQuery CommittedTriangleFrontFace](#rayquery-committedtrianglefrontface)
  - [Payload access qualifiers](#payload-access-qualifiers)
    - [Availability](#availability)
    - [Payload size](#payload-size)
    - [Syntax](#syntax)
    - [Semantics](#semantics)
    - [Detailed semantics](#detailed-semantics)
      - [Local working copy](#local-working-copy)
      - [Shader stage sequence](#shader-stage-sequence)
    - [Example](#example)
    - [Guidelines](#guidelines)
    - [Optimization potential](#optimization-potential)
    - [Advanced examples](#advanced-examples)
      - [Various accesses and recursive TraceRay](#various-accesses-and-recursive-traceray)
      - [Payload as function parameter](#payload-as-function-parameter)
      - [Forwarding payloads to recursive TraceRay calls](#forwarding-payloads-to-recursive-traceray-calls)
      - [Pure input in a loop](#pure-input-in-a-loop)
      - [Conditional pure output overwriting initial value](#conditional-pure-output-overwriting-initial-value)
    - [Payload access qualifiers in DXIL](#payload-access-qualifiers-in-dxil)
- [DDI](#ddi)
  - [General notes](#general-notes)
    - [Descriptor handle encodings](#descriptor-handle-encodings)
  - [State object DDIs](#state-object-ddis)
    - [State subobjects](#state-subobjects)
      - [D3D12DDI_STATE_SUBOBJECT_TYPE](#d3d12ddi_state_subobject_type)
      - [D3D12DDI_STATE_SUBOBJECT_0054](#d3d12ddi_state_subobject_0054)
      - [D3D12DDI_STATE_SUBOBJECT_TYPE_SHADER_EXPORT_SUMMARY](#d3d12ddi_state_subobject_type_shader_export_summary)
      - [D3D12DDI_FUNCTION_SUMMARY_0054](#d3d12ddi_function_summary_0054)
      - [D3D12DDI_FUNCTION_SUMMARY_NODE_0054](#d3d12ddi_function_summary_node_0054)
      - [D3D12_EXPORT_SUMMARY_FLAGS](#d3d12_export_summary_flags)
      - [State object lifetimes as seen by driver](#state-object-lifetimes-as-seen-by-driver)
        - [Collection lifetimes](#collection-lifetimes)
        - [AddToStateObject parent lifetimes](#addtostateobject-parent-lifetimes)
    - [Reporting raytracing support from the driver](#reporting-raytracing-support-from-the-driver)
      - [D3D12DDI_RAYTRACING_TIER](#d3d12ddi_raytracing_tier)
- [Potential future features](#potential-future-features)
  - [Traversal shaders](#traversal-shaders)
  - [More efficient acceleration structure builds](#more-efficient-acceleration-structure-builds)
  - [Beam tracing](#beam-tracing)
  - [ExecuteIndirect improvements](#executeindirect-improvements)
    - [DispatchRays in command signature](#dispatchrays-in-command-signature)
    - [Draw and Dispatch improvements](#draw-and-dispatch-improvements)
    - [BuildRaytracingAccelerationStructure in command signature](#buildraytracingaccelerationstructure-in-command-signature)
- [Change log](#change-log)

---

# Intro

このドキュメントでは、計算とグラフィックス（ラスタライズ）のファーストクラスのピアとして、D3D12 におけるレイトレーシングのサポートについて説明します。ラスタライゼーションパイプラインと同様に、レイトレーシングのパイプラインは、アプリケーションの表現力を最大化するためのプログラマビリティと、ワークロードを効率的に実行するための実装の機会を最大化するための固定機能との間のバランスを取ります。

---

# Overview

システムは、実装が独立してレイを処理できるように設計されています。これには、（これから説明する）さまざまなタイプのシェーダが含まれ、シェーダは単一の入力光線しか見ることができず、飛行中の他の光線の処理順序を見たり、依存したりすることはできない。シェーダの種類によっては、特定の呼び出しの間に複数のレイを生成することができ、必要であればレイの処理結果を見ることができます。いずれにせよ、飛行中に生成されたレイは、決して互いに依存し合うことはあり ません。

このレイの独立性が、並列処理の可能性を広げます。実行中にこれを利用するために、典型的な実装では、スケジューリングと他のタスクの間でバランスを取ることになります。

![raytracing flow](images/raytracing/flow.png)

(上の図は、ある実装がどのようなことをするのかの緩やかな近似に過ぎません。あまり深く読まないでください)。

実行のスケジューリング部分はハードウエア化されているか、少なくともハードウエアに合わせてカスタマイズできる不透明な方法で実装されています。これは典型的には、スレッド間の一貫性を最大化するために、仕事をソートするような戦略を採用するでしょう。API の観点からは、レイスケジューリングはビルトイン機能です。

レイトレーシングの他のタスクは、固定関数と完全にあるいは部分的にプログラマブルな作業の組み合わせです。

最大の固定関数タスクは、アプリケーションによって提供されたジオメトリから構築された acceleration structure をトラバースし、潜在的なレイの交差を効率的に見つけることを目的としています。三角形の交差も固定関数でサポートされています。

シェーダは、いくつかの領域でアプリケーションのプログラマビリティを公開します。

- 暗黙のジオメトリに対する交差の決定（固定関数の三角形交差オプションとは対照的に）

- レイの交差の処理（サーフェスシェーディングなど）またはミスマッチの処理

- 実装に依存しない

また、アプリケーションは、任意の状況でシェーダのプールのうちどれを実行するかを正確に制御し、各シェーダ呼び出しがアクセスするテクスチャなどのリソースに柔軟性を持たせることができます。

---

# Design goals

- 単一のプログラミングモデルにより、レイトレーシング専用のアクセラレーションを持つハードウェアと持たないハードウェアをサポートします。

  - ハードウェア性能の予想される差異は、必要であれば、きれいな機能進行で捕捉されます。

  - 関連する D3D12 のパラダイムを取り入れる。

- アプリケーションは、シェーダのコンパイル、メモリリソース、および全体の同期を明示的に制御することができます。

  - アプリケーションはレイトレーシングをコンピューティングとグラフィックスに緊密に統合することができる

  - インクリメンタルに採用可能

  - PIX のようなツールに友好的

- API キャプチャ/プレイバックなどの実行中のツールは、レイトレーシングをサポートするために不必要なオーバーヘッドを発生させない

  - ローカルルートシグネチャ。その引数は後述のシェーダテーブルから取得し、各シェーダが固有の引数を持つことができる。

---

# Walkthrough

以下のウォークスルーは、この機能のほとんどの構成要素を大まかにカバーし ています。さらなる詳細は、API や HLSL の詳細を記載した専用のセクションを含めて、このド キュメントの後半で説明します。

---

## Initiating raytracing

まず、レイトレーシング シェーダを含むパイプライン・ステートが、SetPipelineState1() によってコマンドリスト上に設定されなければなりません。

そして、ラスタライズが Draw() によって、コンピュートが Dispatch() によって呼び出されるように、レイトレーシングは DispatchRays() によって呼び出されます。DispatchRays() は graphics コマンドリスト、compute コマンドリスト、bundle から呼び出すことができます。

[Tier 1.1](#d3d12_raytracing_tier) implementations also support GPU initiated DispatchRays() via [ExecuteIndirect()](#executeindirect).

[Tier 1.1](#d3d12_raytracing_tier) implementations also support a variant of raytracing that can be invoked from any shader stage (including compute and graphics shaders), but does not involve any other shaders - instead processing happens logically inline with the calling shader. See [Inline raytracing](#inline-raytracing).

---

## Ray generation shaders

[DispatchRays()](#dispatchrays) invokes a grid of ray generation shader invocations.
Each invocation (thread) of a ray generation shader knows its location
in the overall grid, can generate arbitrary rays via TraceRay(), and
operates independently of other invocations. So there is no defined
order of execution of threads with respect to each
other.

HLSL の詳細はこちらです。

---

## Rays

レイとは、原点、方向、パラメトリックな区間 (TMin, TMax) で、区間に沿って T の位置で交差が起こりうるものです。具体的には、レイに沿った位置は、origin + T\*direction (direction は正規化されません) となります。

[レイ範囲](#ray-extents)で定義されたジオメトリタイプによって、カウントする交差点に (TMin..TMax) と [TMin...TMax] という正確な境界があり、多少のニュアンスがあります。

レイは、ユーザー定義のペイロードを伴っており、レイがシーン内のジオメトリと相互作用する際に変更可能で、またその戻り時に [TraceRay()](#traceray) の呼び出し元が見ることができます。[インライン レイトレーシング](#inline-raytracing)の場合、ペイロードは明示的な実体ではなく、[RayQuery::TraceRayInline()](#rayquery-tracerayinline) の呼び出し元がその実行範囲に持つユーザー変数の一部に過ぎません。

![ray](images/raytracing/ray.png)

システムによって追跡される TMin 値は、レイのライフタイム中、決して変化しません。一方、（任意の空間的順序で）交差点が発見されると、システムは TMax を減らして、これまでのところ最も近い交差点を反映させます。すべての交差点が完了すると、TMax は最も近い交差点を表し、その関連性は後述します。

---

## Raytracing output

レイトレーシングでは、シェーダーは明示的に UAV を通じて、(イメージに対するカラーサンプルといった)結果を出力する。

---

## Ray-geometry interaction diagram

今後のセクションでは、この図に加えて、「Miss シェーダ」のようなジオメトリに特化しない示されていない概念について説明します。

![rayGeometryIntersection](images/raytracing/rayGeometryInteraction.png)

---

## Geometry and acceleration structures

シーンのジオメトリは、2 つのレベルの acceleration structure (アクセラレーション・ストラクチャー)を使ってシステムに記述される。bottom-level acclection structure (ボトムレベル・アクセラレーション・ストラクチャー) は、それぞれシーンのビルディングブロックであるジオメトリのセットで構成されています。top-level acceleration structure (トップレベル・アクセラレーション・ストラクチャー)は、bottom-level acceleration structure のインスタンスの集合を表している。

ある Bottom-Level Acceleration Structure には、（1）三角メッシュ、または（2）軸合わせバウンディングボックス（AABB）のみで初期記述された手続き型プリミティブをいくつでも入れることができます。Bottom-Level Acceleration Structure には、1 つのジオメトリタイプしか含めることができません。これらのジオメトリタイプについては、後で詳しく説明します。これらのジオメトリのセットの定義（D3D12_RAYTRACING_GEOMETRY_DESC の配列を介して）が与えられると、アプリケーションは CommandList で BuildRaytracingAccelerationStructure() を呼び、それを表す不透明なアクセラレーション構造をアプリ ケーションが所有する GPU メモリに構築するようにシステムに要求しま す。このアクセラレーション構造は、システムがジオメトリと光線を交 差させるために使用するものです。

Bottom-Level Acceleration Structure のセットが与えられると、アプリケー ションはインスタンスのセットを定義します（GPU メモリに存在する D3D12_RAYTRACING_INSTANCE_DESC 構造を指すことによって）。各インスタンスは Bottom-Level Acceleration Structure を指し、インスタンスを特化するための他の情報を含んでいます。インスタンス定義に含まれる特殊化情報の例としては、行列変換 （インスタンスを世界に配置する）、およびユーザー定義の InstanceID（シェー ダーに固有のインスタンスを識別する）があります。

インスタンスは、この仕様では、わかりやすくするためにジオメトリインスタンスと呼ばれることがあります。

このジオメトリインスタンス定義のセットは（BuildRaytracingAccelerationStructure()を介して）実装に与えられ、アプリケーションが所有する GPU メモリに不透明なトップレベルのアクセラレーション構造を生成し ます。このアクセラレーション構造は、システムが光線をトレースするものを表します。

アプリケーションは、入力リソースとして関連するシェーダにバインドして、複数のトップレベルのアクセラレーション構造を同時に使用できます（HLSL の RaytracingAccelerationStructure を参照）。これにより、必要に応じて、特定のシェーダで異なるジオメトリのセットに光線をトレースすることができます。

> The two level hierarchy for geometry lets applications strike a balance
> between intersection performance (maximized by using larger bottom-level
> acceleration structures) and flexibility (maximized by using more,
> smaller bottom-level acceleration structures and more instances in a
> top-level acceleration structure).

ルールと決定論の議論については、アクセラレーション構造のプロパティを参照してください。

---

## Acceleration structure updates

アプリは BuildRaytracingAccelerationStructure() の D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS を通じて、acceleration structure を更新可能にするよう要求したり、更新可能な acceleration structure への更新を要求したりすることができ ます。

更新可能な acceleration structure（更新前と更新後）は、静的な acceleration structure を一から 構築するほどレイトレーシング性能の点では最適ではありません。しかし、アップデートは、ゼロからアクセラレーション構造を構築するよりも高速になります。

アップデートを実行するために、アプリが変更できる内容には制約があります。例えば、Bottom-Level Acceleration Structure における三角形のジオメトリは、頂点の位置しか更新できません。トップレベルのアクセラレーション構造では、より自由な更新が可能である。詳しくは、acceleration structure 更新の制約を参照してください。

---

## Built-in ray-triangle intersection - triangle mesh geometry

前述のとおり、Bottom-Level Acceleration Structure 内のジオメトリは、ビルトインのレイ・トライアングル・インターセクションのサポートを使用する三角メッシュとして表現することができ、その交差を記述する三角形のバリケンを後続のシェーダに渡します。

---

## Intersection shaders - procedural primitive geometry

Bottom-Level Acceleration Structure 内のジオメトリの代替表現としては、手続き型プリミティブを含む軸合わせされたバウンディングボックスがあります。サーフェスは、レイがバウンディングボックスに当たったときに交差を評価するために、アプリケーション定義の交差シェーダを実行することによって定義されます。シェーダは、現在の T 値を含む、後続のシェーダに渡す交差点を記述する属性を定義します。

ビルドインのレイ-トライアングル交差の代わりに交差シェーダを使用すると、効率は悪くなりますが、はるかに高い柔軟性を提供します。

HLSL の詳細はこちらです。

---

### Minor intersection shader details

交差シェーダは冗長に実行されることがあります。アクセラレーション構造で遭遇する特定の手続き的プリミティブに対して、あるレイに対して交差点シェーダが一度だけ実行される、という保証はどこにもありません。与えられたレイとプリミティブに対して複数回実行することは冗長（無駄）ですが、実装が何らかの理由でトレードオフの価値があると信じるならば、この動作をすることは自由です。このことは、アプリが交差点シェーダから UAV 書き込みを行ったり、特に呼び出しごとに異なる交差点を見つけるような、交差点シェーダへの副作用をオーサリングすることに注意する必要があることを意味します。その結果は、実装によって異なる可能性があります。

あるレイに対して交差点シェーダを複数回呼び出すかどうかに関係なく、実装は常にアプリが選択したジオメトリのフラグを尊重する必要があり、これには `D3D12_RAYTRACING_GEOMETRY_FLAG_NO_DUPLICATE_ANYHIT_INVOCATION` が含まれる場合があります。このフラグを使用すると、任意のヒットシェーダ（次に説明）は、任意のレイ上の任意の交差点に対して一度だけ実行する必要があります。

---

## Any hit shaders

他の交差点に対するレイ上の位置に関係なく、レイが現在のレイの範囲内でジオメトリインスタンスと交差するたびに実行されるユニークなシェーダを定義することができます。これは Any Hit シェーダです。

Any hit シェーダは交差点属性を読み、レイペイロードを修正し、ヒットを無視する (IgnoreHit()) こと、ヒットを受け入れて (実行を終了して) 継続すること、ヒットを受け入れてさらに交差点の検索を停止する (AcceptHitAndEndSearch()) ことを指示することが可能です。

レイパスに沿った交差点に対する Any Hit シェーダの実行順序は定義され ていません。Any hit シェーダがヒットを受け入れると、その T 値は新しい TMax になります。したがって、交差点が見つかる順序によって、他のすべてが同じであれば、異なる数の任意のヒットシェーダの呼び出しが発生します。

システムは、与えられたレイに対して複数の Any Hit シェーダを同時に実行することはでき ません。そのため、Any Hit シェーダは、他のシェーダとの競合を心配すること なく、自由にレイのペイロードを変更することができます。

Any hit シェーダは、例えば、ジオメトリに透明性があるときに便利です。特定のケースは、影の決定における透明性です。任意のヒットシェーダが現在のヒット位置が不透明であることを発見した場合、このヒットを取るが、それ以上の交差を探すのをやめるようにシステムに伝えることができます（単にレイのパス内の何かを探すだけです）。しかし、多くの場合、any hit シェーダは必要なく、ある程度の実行効率が得られ ます。与えられたジオメトリインスタンスに対して、現在のレイ間隔内に交差点 T を持つ Any Hit シェーダがない場合、実装は単に交差点を受け入れ、現在のレイ間隔の TMax を T に減少させます。

これから説明する他のいくつかのシェーダタイプとは異なり、エニーヒットシェーダは新しいレイをトレースすることができません、なぜならここでそうすると、システムの作業が不当に爆発することになるからです。

HLSL の詳細はこちらです。

---

## Closest hit shaders

インスタンス内の各ジオメトリに対して、レイの範囲内で最も近い交差を生成する場合に実行する、ユニークなシェーダを定義することができます。これは、クローズヒットシェーダです。

Closeest Hit シェーダは、交差点属性を読み、レイペイロードを修正し、追加のレイを生成することができます。

> A typical use of a closest hit shader would be to evaluate the color of
> a surface and either contribute to the ray payload or store data to
> memory (via UAV).

レイのパスに沿ったヒットシェーダがある場合は、すべて最接近ヒットシェーダの前に実行されます。特に、最も近いヒットの T 値で両方のシェーダタイプがジオメトリインスタンスに定義されている場合、任意のヒットシェーダは常に最も近いヒットシェーダの前に実行されます。

HLSL の詳細はこちらです。

---

## Miss shaders

どのジオメトリとも交差しないレイについては、ミス・シェーダを指定することができます。ミス・シェーダはレイ・ペイロードを修正し、追加のレイを生成することができます。交差がなかったので、利用可能な交差アトリビュートはありません。

HLSL の詳細はこちらです。

---

## Hit groups

ヒットグループは、以下からなる 1 つまたは複数のシェーダです。{0 または 1 交差点シェーダ、0 または 1 任意のヒットシェーダ、0 または 1 最接近ヒットシェーダ}で構成される 1 つ以上のシェーダです。あるインスタンスの個々のジオメトリは、それぞれヒットグループを参照し、シェーダコードを提供します。グループ化のポイントは、光線がジオメトリと相互作用するときに、実装が効率的にグループをコンパイルして実行できるようにすることです。

光線生成シェーダとミスシェーダはジオメトリに直接関与しないので、ヒッ トグループの一部ではありません。

ヒットグループに交差シェーダが含まれている場合、プロシージャルのプリミティブ ジオメトリに対してのみ使用することができます。ヒットグループに交差点シェーダが含まれていない場合、三角形ジオメトリにのみ使用できます。

シェーダを全く含まないヒットグループも可能で、その場合はシェーダ識別子に NULL を使用します（コンセプトは後述）。 これは不透明なジオメトリとしてカウントされます。

> An empty hit group can be useful, for example, if the app doesn't want
> to do anything for hits and only cares about the [miss shader](#miss-shaders) running when nothing has been hit.

---

## TraceRay control flow

シェーダが TraceRay()を呼び出すと、次のようなことが起こります。

![traceRayControlFlow](images/raytracing/traceRayControlFlow.png)

[1] この段階では、acceleration structure を検索し、レイと交差する可能性のあるプリミティブを列挙します（保守的）。プリミティブがレイと交差し、現在の[レイの範囲](#ray-extents)内にある場合、それは最終的に列挙されることが保証されています。プリミティブがレイと交差しないか、[レイ範囲](#ray-extents)の外側にある場合、列挙されるかどうかわかりません。TMax はヒットがコミットされるときに更新されることに注意してください。

[2] 交差点シェーダが実行中に ReportHit() を呼び出すと、後続のロジックが交差点を処理し、 [5] を経由して交差点シェーダに戻ります。

[3] 不透明度は、レイフラグと同様に交差点のジオメトリとインスタンスフラグを調べることによって決定されます。また、任意のヒットシェーダがない場合、ジオメトリは不透 明とみなされます。

[4] `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH` [レイフラグ](#ray-flags)が設定されているか、あるいは Any Hit シェーダが AcceptHitAndEndSearch() を呼び出し、 AcceptHitAndEndSearch() コールサイトでの Any Hit シェーダの実行が中断されると、この時点でヒットの検索は終了 します。少なくともこのヒットがコミットされたので、もし存在すれば(そして `RAY_FLAG_SKIP_CLOSEST_HIT_SHADER` によって無効になっていなければ)、これまでに最も近いヒットはどれでもその上で実行されま す。

[5] 交差したプリミティブが三角形でない場合、交差シェーダはまだア クティブで、ReportHit()の呼び出しを含む可能性があるため、実行を 再開します。

---

## Flags per ray

[TraceRay()](#traceray) supports a selection of [ray flags](#ray-flags) to override transparency, culling, and early-out
behavior.

> To illustrate the utility of ray flags, consider how they would help
> implement one of multiple approaches to rendering shadows. Suppose an
> app wants to trace rays to distant light sources to accumulate light
> contributions for rays that don't hit any geometry, using tail
> recursion.
>
> [TraceRay()](#traceray) could be called with
> `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER` flags from the [ray generation shader](#ray-generation-shaders),
> followed by exiting the shader withn nothing else to do. Any hit shaders,
> if present on geometry, would execute to determine transparency,
> though these shader invocations could be skipped if desired by also including `RAY_FLAG_FORCE_OPAQUE`.
>
> If any geometry hit is encountered (not necessarily the closest hit),
> ray processing stops, due to
> `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH`. A hit has been
> committed/found, but there is no [closest hit shader](#closest-hit-shaders) invocation, due to
> `RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`. So processing of the ray ends
> with no action.
>
> Rays that don't hit anything cause the [miss shader](#miss-shaders) to
> run, where light contribution is evaluated and written to a UAV. So in
> this scenario, geometry in the acceleration structure acted to cull miss
> shader invocations, ignoring every other type of shader (unless needed
> for transparency evaluation).
>
> Skipping shaders can alternatively be accomplished by setting shader
> bindings to NULL (shader bindings details are discussed [later on](#shader-identifier)).
> But the use of ray flags in this example means the implementation doesn't
> even have to look up shader bindings (only to find that they are NULL).
> Which also means the app doesn't have to bother configuring NULL bindings
> anywhere.

---

## Instance masking

[Geometry instances](#d3d12_raytracing_instance_desc) in top-level
acceleration structures each contain an 8-bit user defined InstanceMask.
[TraceRay()](#traceray) has an 8-bit input parameter
InstanceInclusionMask which gets ANDed with the InstanceMask from any
geometry instance that is a candidate for intersection. If the result of
the AND is zero, the intersection is ignored.

> This feature allows apps to represent different subsets of geometry
> within a single acceleration structure as opposed to having to build
> separate acceleration structures for each subset. The app can choose how
> to trade traversal performance versus overhead for maintaining multiple
> acceleration structures.
>
> An example would be culling objects that an app doesn't want to
> contribute to a shadow determination but otherwise remain visible.
>
> Another way to look at this is:
>
> The bits in InstanceMask define which "groups" an instance belongs to.
> (If it is set to zero the instance will always be rejected\!)
>
> The bits in the ray's InstanceInclusionMask define which groups to
> include during traversal.

---

## Callable shaders

Callable シェーダは、実行効率を多少犠牲にして、病的なシェー ダの並べ替えやシェーダネットワークを支援することを意図しています。

Callable シェーダは、後述する[シェーダーテーブル](#shader-tables)を通じて定義され ますが、基本的にはユーザ定義関数テーブルです。このテーブルは GPU 仮想アドレス（D3D12_DISPATCH_RAYS_DESC の CallableShaderTable）を DispatchRays() コールに与えることによって識別されます。テーブルの内容には、GetShaderIdentifier() によってステートオブジェクト（後述）から取得されたシェーダ識別子が含まれています。

与えられた callable シェーダは、（HLSL の CallShader() を介して）シェーダテーブルにインデックスを付け、任意のレイトレー シングシェーダからどの callable シェーダを呼び出すかを選択することによって呼び出され ます。呼び出しを行うシェーダの呼び出しは、任意の in/out パラメータを持つサブルーチン呼び出しのように、呼び出し可能なシェーダの 1 つの呼び出しを生成するだけです。したがって、呼び出しが戻ると、呼び出し元は期待通りに続行します。呼び出し可能なシェーダは他のシェーダとは別にコンパイルされるため、コンパイラは、合意された関数シグネチャ以外に呼び出し側/呼び出し側についていかなる仮定も立てることができません。実装は、（レジスタを介して渡すことを決定しなかった）パラメータやライブステートを格納するために、ユーザ定義の最大サイズのスタックを使用する方法を選択します。

実装では、呼び出し可能なシェーダを呼び出し側のシェーダとは別に実行するようスケジューリングすることが期待されています。これは、光線を追跡すると、他のシェーダが実行されるのと同様です。したがって、小さなプログラムを実行するためにこの機能を使 うことは、シェーダをスケジューリングして実行する最小限のオーバーヘッド に見合わないかもしれません。

> In the absence of callable shaders as a feature, applications could
> achieve the same result by tracing rays with a NULL acceleration
> structure, which causes a miss shader to run, repurposing the ray
> payload and potentially the ray itself as function parameters. Except
> doing this miss shader hack would be wasteful in terms of defining a ray
> that is guaranteed to miss for no reason. Rather than supporting this
> hack, callable shaders are seen as a cleaner equivalent.
>
> The bottom line is implementations should not have difficulty supporting
> callable shaders given the system has to support miss shaders anyway. At
> the same time, apps must not expect execution efficiency that would
> greatly exceed that of invoking a miss shader from a raytrace (minus the
> actual ray processing overhead).

---

## Resource binding

光線はどこにでも行くことができるので、レイトレーシングでは、シーンの すべてのシェーダが同時に実行可能であるだけでなく、それらのリソースバ インディングも利用可能でなければなりません。実際、実行するシェーダの選択（後述のシェーダ識別子による）は、従来のルート署名バインディング（記述子テーブル、ルート記述子、ルート定数）と共に、単なるリソースバインディングとみなされます。

SetDescriptorHeaps()によって CommandList に設定された記述子ヒープ は、レイトレーシング、グラフィックス、および計算で共有されます。

---

### Local root signatures vs global root signatures

レイトレーシング用シェーダでは、バインディングは以下のルート署名の一方または両方によって定義することができます。

- グローバルルートシグネチャ。その引数はすべてのレイトレーシング用シェーダで共有され、コマンドリストの PSO を計算し、SetComputeRootSignature()（または同等の間接状態設定 API が存在すればそれ）を介して設定されます。

- シェーダから呼び出されるライブラリ関数にはコード定義が必要です。

一緒に使用される各レイトレーシングシェーダは、異なるローカルルー トシグネチャを使用できますが、同じグローバルルートシグネチャを使用しなけ ればなりません。グローバル」ルート署名は、コマンドリストの計算状態に使用されるルート 署名と同じです。

DispatchRays()呼び出し（または同等の間接 API が存在する場合）中に呼び出されるシェーダが、上記のように CommandList に設定された同じグローバルルート署名を使用する限り、 State オブジェクト（後述）に一緒に集められたシェーダの異なるセットは、異なるグローバル ルート署 名を持つことができます。

グローバルルートシグネチャとは異なり、ローカルルートシグネチャは保持できるエントリ数に大きな制限があり、これはサポートされる最大シェーダレコードストライドの 4096 バイトから、シェーダ識別子のサイズとして 32 バイトを引いた 4064 バイトが最大ローカルルートシグネチャフットプリントとして境界となっていま す。シェーダ識別子とシェーダレコードについては、以下でさらに説明します。

ローカル・ルート・シグネチャで指定されたシェーダの「レジスタ」バインディ ング（たとえば、t0、u0 など）は、あるシェーダのグローバル・ルート・シグネチャ のものと重なることはありません。

スタティックサンプラーに関する注記。ただし、レイトレーシング・パイプライン・ステート・オブジェクト（後述）で使用される各ローカル・ルート署名は、同じものを定義する他のローカル・ルート署名と同じように、使用するすべてのスタティック・サンプラーを定義する必要があります。つまり、ローカルルートシグネチャが例えばサンプラー s0 の定義を行う場合、s0 を定義するすべてのローカルルートシグネチャは同じ定義を使用しなければなりません。さらに、ローカルルートシグネチャとグローバルルートシグネチャにまたがるユニークなスタティックサンプラーの合計数は、D3D12 のリソースバインドモデルのスタティックサンプラー制限に収まる必要があります。

> The reason that local root signatures must not have any conflicting
> static sampler definitions is to enable shaders to be compiled
> individually on implementations that have to emulate static samplers
> using descriptor heaps. Such implementations can pick a fixed location
> in a sampler descriptor heap to place a static sampler, knowing that
> other shaders that might use a different local root signature and define
> the same sampler will use the same slot. Static samplers in the global
> root signature can also be handled the same way (given that as mentioned
> above, register bindings can't overlap across global and local root
> signatures.

ルートシグネチャのシェーダ visibility フラグについては、[ここ](#note-on-shader-visibility)で説明しています。

---

## Shader identifier

シェーダー識別子は 32 バイトの不透明なデータブロックで、レイトレー シングシェーダー（レイ生成シェーダー、ヒットグループ、ミスシェーダー、 コール可能シェーダー）を（現在のデバイス／プロセス内で）一意に識別し ます。アプリケーションは、これらのシェーダーのどれでも、システムからシェーダー識別子を要求することができます。これは、シェーダへのポインタと考えることができます。

レイトレーシング・プロセスが実行するシェーダを探すときに、アプリから NULL シェーダ識別子に遭遇した場合、その目的のためのシェーダは実行されず、レイトレーシング・プロセスは継続されます。ヒットグループの場合、NULL シェーダ識別子は単に、それが含むどのタイプのシェーダも実行されないことを意味します。

アプリケーションは同じシェーダを複数回作成することがあります。これは同じコードでもエクスポート名が同じだったり、違っ たり、別々のレイトレーシング・パイプラインやコードのコレクション（後述） にまたがっている可能性があります。この場合、一見同じに見えるシェーダは、実装によって同じ識別子を返すことも返さないこともあります。いずれにせよ、実行動作は指定されたシェーダコードと一致します。

---

## Shader record

```
shader record = {shader identifier, local root arguments for the shader}
```

シェーダレコードとは、上記のレイアウトにおいて、アプリケーションが所有するメモリ領域を指すだけである。アプリケーションは任意のレイトレーシング用シェーダのシェーダ識別子を取得することができるため、任意の方法で任意の場所にシェーダレコードを作成することができます。シェーダがローカル・ルート・シグネチャを使用する場合、そのシェーダ・レコードにはそのルート・シグネチャ用の引数が含まれます。シェーダーレコードの最大ストライドは 4096 バイトです。

---

## Shader tables

```
shader table = {shader record A}, {shader record B} ...
```

シェーダーテーブルは、メモリの連続した領域内のシェーダーレコードのセットです。開始アドレスは 64 バイトにアラインされている必要があります（D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT）。

レイトレーシングはシェーダーテーブルに（さまざまな方法で）インデックスを作 成し、シーンの異なる部分すべてに対して固有のシェーダーとリソースバ インディングを実行できるようにします。アクセスされる特定のシェーダーレコードのみが、有効に入力される必要があります。

シェーダーテーブルのための API オブジェクトはなく、アプリは単にメモリ内の領域をシェーダーテーブルであると識別するだけです。むしろ、DispatchRays() のパラメータは、アプリが（とりわけ）以下のタイプのシェーダーテーブルを識別するためのメモリへのポインタを含んでいます。

- [ray generation shader](#ray-generation-shaders) (single entry since only one shader record is
  needed)

- [hit groups](#hit-groups)

- [miss shaders](#miss-shaders)

- [callable shaders](#callable-shaders)

---

## Indexing into shader tables

与えられたジオメトリの交差点で使用する適切なシェーダを見つけるためのシェーダテーブル内の位置は、柔軟性を持たせるために、アプリケーションによって異なる場所で提供されるさまざまなオフセットの合計として計算されます。

詳細はシェーダーテーブル内のアドレス計算で説明しますが、基本的にはシェーダーテーブルのベースアドレスとレコードストライドを提供する DispatchRays()で処理が開始されます。次に、各ジオメトリとレイトレーシング アクセラレーション構造内の各ジオメトリ インスタンス定義がインデキシングに値を提供します。そして、最後の貢献はシェーダ内の TraceRay() 呼び出しによって提供され、ジオメトリ/インスタンスまたはアクセラレーション構造自体を変更することなく、与えられたジオメトリインスタンスで使用するシェーダと引数（バインディング）をさらに差別化することを可能にします。

---

### Shader record stride

アプリケーションは、DispatchRays()のパラメータとして、システムがレコードに使用することを望むデータストライドを指定します。すべてのシェーダーテーブルのインデックス演算は、このレコードストライドの倍数として行われます。これは、サイズ [0...4096] バイトの 32 バイト (D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT) の任意の倍数とすることができます。

stride が非ゼロの場合、stride は少なくとも最大のシェーダーレコードと同じ大きさでなければならない。したがって、シェーダーレコードがストライドより小さい場合、シェーダーレコードの間に未使用のメモリが存在することになる。

stride が 0 の場合、すべてのインデックス付けは同じシェーダーレコードを指す。これは特に、明示的なグローバルルートシグネチャと冗長にローカルルートシグネチャをグローバルな方法で動作させることを考えると、興味深いことではなさそうです。しかし、これはテストやデバッグをするには便利かもしれません。

---

### Shader table memory initialization

システムがストライドを使用してシェーダテーブルにインデックスを作 成し、レコードに到達すると、有効なシェーダ識別子があり、その後に適切な量のロー カルルート引数が続く必要があります。個々のローカル・ルート引数は、シェーダーの実行がそれらを参照する場合にのみ初期化する必要があります。

シェーダーテーブルのあるレコードでは、シェーダー識別子に続くルート引数は、指定されたシェーダーがコンパイルされたときのローカルルート署名と一致しなければなりません。引数のレイアウトは、各引数をその個別の（定義された）サイズに揃えるために必要なパディングで、ローカル・ルート・シグネチャで宣言された順序で詰めることによって定義されます。たとえば、ルート記述子および記述子ハンドル（記述子テーブルの識別）はそれぞれ 8 バイトのサイズであるため、どの引数が先行しても、レコードの先頭から最も近い 8 バイトにアラインされたオフセットにある必要があります。

---

## Inline raytracing

[TraceRayInline()](#rayquery-tracerayinline) is an alternative to [TraceRay()](#traceray) that doesn't use any separate shaders - all shading is handled by the caller. Both styles of raytracing use the same acceleration structures.

`TraceRayInline()`, as a member of the [RayQuery](#rayquery) object, actually does very little itself - it initializes raytracing parameters. This sets up the shader to call other methods of the [RayQuery](#rayquery) object to work through the actual raytracing process.

シェーダは `RayQuery` オブジェクトをローカル変数としてインスタンス化でき、それぞれがレイ クエリのステートマシンとして動作します。 シェーダは `RayQuery` オブジェクトのメソッドと対話し、アクセラレーション構造体とトラバーサル情報を使ってクエリを進めます。 アクセラレーション構造へのアクセス（例えば、ボックスと三角形の交差）は抽象化されているため、ハードウェアに任されています。 これらの固定機能アクセラレーション構造へのアクセスを囲んで、列挙された候補ヒットとクエリの最終結果（ヒット対ミスなど）の両方を処理するために必要なすべてのアプリシェーダコードは、RayQuery を駆動する個々のシェーダに含めることができます。

`RayQuery` objects can be used in any shader stage, including compute shaders, pixel shaders etc. These can even be using in any raytracing shaders: any hit, closest hit etc., combining both raytracing styles.

インラインレイシングは、Tier 1.1 のレイトレーシング実装でサポートされています。

シュードコードの例は[こちら](#tracerayinline-examples)です。

> The motivations for this second parallel raytracing system are both the any-shader-stage property as well as being open to the possibility that for certain scenarios the full dynamic- shader-based raytracing system may be overkill. The tradeoff is that by inlining shading work with the caller, the system has far less opportunity to make performance optimizations on behalf of the app. Still, if the app can constrain the complexity of its raytracing related shading work (while inlining with other non raytracing shaders) this path could be a win versus spawning separate shaders with the fully general path.
>
> One simple scenario for inline raytracing is tracing rays with the `RayQuery` object initialized with template flags: `RAY_FLAG_CULL_NON_OPAQUE | RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES | RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH`, using an acceleration structure with only triangle based geometry. In this case the system can see that it is only being asked to find either a hit or a miss in one step, which it could potentially fast-path. This could enable basic shadow determination from any shader stage as long as no transparency is involved. Should more complexity in traversal be required, of course the full state machine is available for completeness and generality.
>
> It is likely that shoehorning fully dynamic shading via heavy uber-shading through inline raytracing will have performance that depends extra heavily on the degree of coherence across threads. Being careful not to lose too much performance here may be a burden largely if not entirely for the the application and it's data organization as opposed to the system.

---

### TraceRayInline control flow

The image below depicts what happens when a shader uses a [RayQuery](#rayquery) object to call [RayQuery::TraceRayInline()](#rayquery-tracerayinline) and related methods for performing inline raytracing. It offers similar functionality to the full dynamic-shader-based [TraceRay() control flow](#traceray-control-flow), except refactored to make more sense when driven from a single shader. The orange boxes represent fixed function operations while the blue boxes represent cases where control has returned to the originating shader to drive what happens next, if anything. In each of these states of shader control, the shader can choose to further interact with the `RayQuery` object via a subset of [RayQuery intrinsics](#rayquery-intrinsics) currently valid based on the current state.

![traceRayInlineControlFlow](images/raytracing/traceRayInlineControlFlow.png)

[1][rayquery::proceed()](#rayquery-proceed) searches the acceleration structure to enumerate
primitives that may intersect the ray, conservatively: If a primitive is
intersected by the ray and is within the current [ray extents](#ray-extents)
interval, it is guaranteed to be enumerated eventually. If a primitive
is not intersected by the ray or is outside the current [ray extents](#ray-extents), it may or may not be enumerated. Note that TMax is updated
when a hit is committed. [RayQuery::Proceed()](#rayquery-proceed) represents where the bulk of system acceleration structure traversal is implemented (including code inlining where applicable). [RayQuery::Abort()](#rayquery-abort) is an optional shortcut for the shader to be able to cause traversal to appear to be complete, via [RayQuery::Proceed()](#rayquery-proceed) returning `FALSE`. So it is just a convenient way to exit the shader's traversal loop. A shader can instead choose to break out of its traversal logic manually as well with normal shader code branching (invisible to the system). This works since, as discussed in [5], the shader can call [RayQuery::CommittedStatus()](#rayquery-committedstatus) and related methods for retrieving committed hit information at any time after a [RayQuery::TraceRayInline](#rayquery-tracerayinline) call.

[2] Consider the case where the geometry is not triangle based. Instead of fixed function triangle intersection [RayQuery::Proceed()](#rayquery-proceed) returns control to the shader. It is the responsibility of the shader to evaluate all procedural intersections for this acceleration structure node, including resolving transparency for them if necessary without the system seeing what's happening. The net result in terms of traversal is to call[RayQuery::CommitProceduralPrimitiveHit()](#rayquery-commitproceduralprimitivehit) at most once if the shader finds an opaque hit that is closest so far.

[3] 不透明度は、レイフラグの選択（テンプレートパラメータとダイナミックフラグを OR したもの）と同様に、交差点のジオメトリとインスタンスフラグを調べることによって決定されます。

[4] While in `CANDIDATE_PROCEDURAL_PRIMITIVE` state it is ok to call [RayQuery::CommitProceduralPrimitiveHit()](#rayquery-commitproceduralprimitivehit) zero or more times for a given candidate, as long as each time called, the shader has manually ensured that the latest hit being committed is within the current [ray extents](#ray-extents). The system does not do this work for procedural primitives. So new committed hits will update TMax multiple times as they are encountered. For simplicity, the flow diagram doesn't visually depict the situation of multiple commits per candidate. Alternatively the shader could enumerate all procedural hits for the current candidate and just call `CommitProceduralPrimitiveHit()` once for the closest hit found that the shader has calculated will be the new TMax.

[5] While in `CANDIDATE_NON_OPAQUE_TRIANGLE` state, the system has already determined that the candidate would be the closest hit so far in the [ray extents](#ray-extents) (e.g. would be new TMax if committed). It is ok for the shader to call [RayQuery::CommitNonOpaqueTriangleHit()](#rayquery-commitnonopaquetrianglehit) zero or more times for a given candidate. If called more than once, subsequent calls simply have not effect as the hit has already been committed. For simplicity, the flow diagram doesn't visually depict the situation of multiple commits per candidate.

[ `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH` レイフラグが設定されている場合、ヒットの検索はこの時点で終了します。

[7] The shader can call [RayQuery::CommittedStatus()](#rayquery-committedstatus) from anywhere - a careful look at the flow diagram will reveal this. It doesn't have to be after traversal has completed. The status (including values returned from methods that report commited values) simply reflect the current state of the query. What is not depicted in the diagram is that if [RayQuery::CommittedStatus()](#rayquery-committedstatus) is called before traversal has completed, the shader can still continue with the ray query. One scenario where it can be interesting to call [RayQuery::CommittedStatus()](#rayquery-committedstatus) before [RayQuery::Proceed()](#rayquery-proceed) has returned false is if the shader has chosen to manually break out of its traversal loop without calling [RayQuery::Abort()](#rayquery-abort) discussed in [1].

[8] The endpoint of the graph is trivially reachable anytime the shader has control simply by not calling methods that advance `RayQuery` state. The shader can arbitrarily choose to stop using the `RayQuery` object, or do final shading based on whatever the current state of the `RayQuery` object is. The shader can even reset the query regardless of its current state at any time by calling [RayQuery::TraceRayInline()](#rayquery-tracerayinline) again to initialize a new trace.

### Specialized TraceRayInline control flow

下の図は、 `RayQuery` の最初の宣言で使用されるテンプレートフラグの特定の選択が、フルフローグラフ（上の図）をどのようにプルダウンすることができるかを示しています。 ここでは、交差の検索にシェーダが参加する必要はありません。 さらに、検索は最初のヒットで終了するように設定されています。 このような簡略化により、システムは、より性能の高いインラインレイシングコードを生成するために解放されます。

![traceRayInlineControlFlow2](images/raytracing/traceRayInlineControlFlow2.png)

---

# Shader management

---

## Problem space

---

### Implementations juggle many shaders

CommandList から DispatchRays() を呼び出すと、光線はどこにでも行くことができるので、アプリケーションは呼び出されるかもしれないすべてのシェーダを指定する方法を持たなければなりません。これは、アプリケーションが任意にシェーダとそのルート引数を選択できるシェーダテーブルの存在によって解決されるように思われます。

しかし、実装は、もし彼らが前もって（実行前に）完全なセットを見る機会を得 るなら、より効率的にシェーダの任意のセットを実行する可能性を持っています。そこで、設計上の選択は、実装にクイックリンクのステップを実行する能力を与えることです。このリンクは個々のシェーダを再コンパイルするのではなく、参照される可能性のあるセット内のすべてのシェーダの特性に基づいて、いくつかのスケジューリング決定を行います。アプリケーションに自由があるのは、シェーダテーブルのどこからでも、指定された事前特定セット内のシェーダを参照できるところです。

シェーダのセットが事前に定義されている必要があるのは、ある DispatchRays()コールで到達可能なセットを把握するために、ドライバにシェーダテーブルを検査させることが実行不可能なためです。シェーダテーブルは GPU のタイムライン上でアプリケーションによって （適切なステートバリアによって）自由に変更することができます。期待されるのは、グループのスケジューリング最適化のためにシェーダの セットを分析することは、ドライバの CPU タスクとして残すのが最善で あるということです。

このことは、CPU タイムライン上でレイトレーシング操作によって到達可能なすべてのシェーダの何らかの表現を作成する必要性を動機づけます。

---

### Applications control shader compilation

特に大規模なアセットベースの場合、CPU コストが高いため、アプリケー ションはいつ、どこで（どのスレッドで）シェーダのコンパイルが行われるかを 制御する必要があります。

まず、DXIL バイナリへの最初のシェーダコンパイルがあります。これは、アプリケー ションによってオフラインで（ハードウェアドライバがそれを見る前に）行うことができま す。HLSL コンパイラは DXIL ライブラリをサポートしており、アプリケーションは、必要に応じて、大きなコンパイル済みコード ベースを単一ファイルに簡単に格納することができます。

1 つまたは複数の DXIL ライブラリにシェーダがある場合、シェーダが実行される任意のシステムでコンパイルするためにドライバに送信する必要があります。アプリケーションは、任意の時点でドライバがコンパイルすべき任意の DXIL ライブラリのサブセットを選択できなければなりません。アプリケーションは、シェーダのグループがどのように DXIL ライブラリにパッケージされるかにかかわらず、ドライバのシェーダ コンパイルをスレッド間で分散する方法を自由に選択することができます。

---

## State objects

状態オブジェクトは、アプリケーションが単一ユニットとして管理する、シェーダを含むさまざまな量の設定状態を表し、ドライバが適切と考える方法で処理（コンパイルや最適化など）するためにアトム的に与えられます。ステートオブジェクトは、D3D12 デバイスの CreateStateObject()で作成されます。

---

### Subobjects

ステートオブジェクトは、サブオブジェクトから構築されます。サブオブジェクトは型と対応するデータを持っています。サブオブジェクトの型の例をいくつか挙げてみましょう。 `D3D12_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY` と `D3D12_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE*` .

もう一つの注目すべきサブオブジェクトのタイプは、 `D3D12_STATE_SUBOBJECT_TYPE_SUBOBJECT_TO_SHADERS_ASSOCIATION` で、その役割は、別のサブオブジェクトを DXIL エクスポートのリストと関連付けることにあります。これにより、例えば、複数のローカルルートシグネチャを同時にステートオブジェクトに存在させ、それぞれを異なるシェーダーエクスポートに関連付けることができます。詳細については、Subobject association behavior を参照してください。また、ステートオブジェクト API の関連する部分については、ここを参照してください。D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION と D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION を参照してください。

サブオブジェクトタイプの完全なセットは、D3D12_STATE_SUBOBJECT_TYPE で定義されています。

---

#### Subobjects in DXIL libraries

ステートオブジェクトの作成前にオフラインでコンパイルされた DXIL ライブラリも、ステートオブジェクトで直接定義できるものと同じ種類のサブオブジェクトの多くを定義することができます。DXIL/HLSL バージョンのサブオブジェクトは[ここ](#subobject-definitions)で定義されます。

サブオブジェクトを DXIL ライブラリまたはステート オブジェクトのいずれかで定義できるのは、オフライン（DXIL ライブラリ）またはランタイム（ステート オブジェクト）でどの程度ステートをオーサリングするか、アプリケーションで選択できるようにするためです。

シェーダはサブオブジェクトとみなされないため、DXIL ライブラリに存在しますが、ステートオブジェクトに直接渡すことはできません。代わりに、シェーダーをステート・オブジェクトに取り込む方法は、含まれている DXIL ライブラリを DXIL ライブラリ・サブオブジェクトとして、取り込むべきシェーダー・エントリーポイント名を含めてステート・オブジェクトに入れることである。

---

### State object types

ステートオブジェクトには、それが含むサブオブジェクトとステートオブジェクトの使用方法に関するルールを指示するタイプがあります。

---

#### Raytracing pipeline state object

ステートオブジェクトのタイプの 1 つは `D3D12_STATE_OBJECT_TYPE_RAYTRACING_PIPELINE` , またはレイトレーシング パイプライン ステート オブジェクト（略して RTPSO）です。RTPSO は、ローカルルートシグネチャやその他の状態など、すべての構成オプションが解決された状態で、DispatchRays コールで到達可能なシェーダのフルセットを表します。

RTPSO は、実行可能なステートオブジェクトと考えることができます。

SetPipelineState1()への入力はステートオブジェクトであり、RTPSO はコマンドリストにバインドされることができます。

---

#### Graphics and compute state objects

将来的には、グラフィックスと計算パイプラインは、完全性のために、ステートオブジェクト形式で定義される可能性があります。当初はレイトレーシングを有効にすることに重点を置いています。そのため、今のところ、グラフィックスおよびコンピュート PSO の構築方法は変更されていません。

---

#### Collection state object

もう 1 つのステートオブジェクトタイプは、 `D3D12_STATE_OBJECT_TYPE_COLLECTION` 、または略してコレクションです。コレクションは任意の量のサブオブジェクトを含むことができますが、制約を持ちません。含まれるサブオブジェクトが持つ依存関係は、すべて同じコレクションで解決されなければならないわけではありません。依存関係がローカルに定義されていても、サブオブジェクトのセットは、最終的に GPU で使用される状態の完全なセットである必要はありません。たとえば、コレクションはシーンをレイトレースするのに必要なすべてのシェーダを 含まないかもしれませんが、それは可能です。

コレクションの目的は、アプリケーションが一度に（たとえば、与えられたスレッドで）コンパイルするためにドライバに状態の任意に大きいまたは小さいコレクションを渡すことができるようにすることです。

コレクション内のサブオブジェクトで提供される設定情報が少なすぎると、ドライバは何もコンパイルできず、単にサブオブジェクトを保存するだけで終わってしまう。 この状況はデフォルトでは許可されません。コレクションの作成は失敗します。 コレクションは、D3D12_STATE_OBJECT_FLAG_ALLOW_LOCAL_DEPENDENCIES_ON_EXTERNAL_DEFINITONS` フラグを設定することにより、全ての依存関係を解決する必要性を回避し、ドライバのコンパイルを遅延させることが可能です。

コレクションは、ドライバがすぐにコンパイルできるように、次の要件を満たしている必要があります。

- シェーダによって参照されるリソース バインディングは、バインディングを定義するローカルおよび/またはグローバル ルート署名サブオブジェクトを持つ必要があります。
- レイトレーシング シェーダーは D3D12_RAYTRACING_SHADER_CONFIG と D3D12_RAYTRACING_PIPELINE_CONFIG サブオブジェク トを持たなければならない。
- 指定された DXIL ライブラリ

上記のリストのうち、サブオブジェクトの関連付けに関係する部分については、サブオブジェクトの関連付けの要件でさらに説明します。

簡単のために、コレクションは他のコレクションから作ることはできません。実行可能なステートオブジェクト（RTPSO など）だけが、既存のコレクションを定義の一部として取り込むことができます。

[State object lifetimes as seen by driver](#state-object-lifetimes-as-seen-by-driver) is a discussion useful for driver authors.

---

#### Collections vs libraries

コレクションはライブラリと少し似ていますが、DXIL ライブラリと区別するために別の名前が使用されます。

DXIL ライブラリはハードウェアにとらわれません。これに対して、コレクションのコンテンツは、処理するドライバに与えられ、DXIL ライブラリの一部（使用するエクスポートのサブセットをリストすることによって）、複数の DXIL ライブラリ、さらにルート署名のような他のタイプのサブオブジェクトを含めることができる。

アプリケーションは、それぞれが単一のコンパイル可能なレイトレーシング シェーダを持つ多数の小さな DXIL ライブラリを持つことを選択することができます。CPU 上の異なるスレッドにまたがって、それぞれのコレクションを作成することができます。または、スレッドごとに 1 つのコレクションを作成し、シェーダのセットをそれらに均等に分散させることができます。いずれの場合も、RTPSO はコレクションのセットから構築されます。

また、使用前にドライバでシェーダーをコンパイルするのにかかる CPU 時間が気にならない場合は、アプリケーションでコレクションの作成をスキップして、すべての DXIL ライブラリを直接 RTPSO の作成に渡すことができます。極端な例として、アプリケーションがすべてのシェーダーアセット（およびルート署名などの必要なサブオブジェクト）を単一の DXIL ライブラリ（1 つのバイナリファイルなど）にベイクし、これをメモリにロードして RTPSO の作成に直接渡すことが考えられます。ドライバは、1 つのスレッドですべてのシェーダを一度にコンパイルする必要があります。

---

### DXIL libraries and state objects example

![librariesAndCollections](images/raytracing/librariesAndCollections.png)

---

### Subobject association behavior

サブオブジェクトの紹介では、別のサブオブジェクトをシェーダエクスポートのセットと関連付ける、特定のタイプのサブオブジェクトを呼びました。

このセクションでは、サブオブジェクト（ルート署名のような）が DXIL ライブラリおよびステートオブジェクトのシェーダとどのように関連付けられるかを説明します。これには、デフォルトの関連付け（便宜上意図されたもの）と明示的な関連付けの動作方法が含まれます。また、アプリが DXIL ライブラリをステートオブジェクトに含める場合、DXIL サブオブジェクトの関連付けをどのように上書きするかも説明します。

---

#### Default associations

デフォルトの関連付けは、特定のサブオブジェクト（ルート署名など）が多くのシェーダーで使用される場合に便利です。

---

##### Terminology

この後の議論では、一連のシェーダを見つけることができる可視性の次のスコープを考えてください。

- コレクション状態オブジェクト（1 つまたは複数の DXIL ライブラリからシェーダを取得することができます。

- 実行可能なステートオブジェクト（RTPSO など）、1 つまたは複数のコレクションや DXIL ライブラリからシェーダを取得することができる

- 与えられたスコープでサブオブジェクトを宣言し、そのスコープでそれを参照する明示的な関連付けがないこと。  このサブオブジェクトが、他のスコープで定義された関連に関与している場合、そのスコープを囲む、または含まれるスコープも含めて、ローカルにこのサブオブジェクトがデフォルトの関連として機能することに影響を与えません。

与えられたスコープは他の内部スコープを含むことができ、また外部スコープはそれを囲むことができます。

---

##### Declaring a default association

シェーダのセットに対するサブオブジェクトのデフォルトの関連付けを宣言する 2 つの方法があります。

1. 空のエクスポートリストを持つ関連付けを定義する。  指定されたサブオブジェクトは、現在のスコープに存在することも、存在しないこともあります。  指定されたサブオブジェクトは、定義されているステートオブジェクトが RTPSO などの実行可能なものでない限り、未解決（現在のスコープ、含むスコープ、囲まれたスコープで定義されていない）である可能性もあります。

2. アクセラレーション構造がビルドされると、アプリのアクセラレーション構造記述によって指される頂点バッファなどを含む、ビルドへの入力への参照を保持しません。

---

##### Behavior of a default association

デフォルトの関連付けでは、サブオブジェクトは、現在のスコープおよび含まれるスコープ内のすべてのエクスポート候補と関連付けられますが、包含するスコープとは関連付けられません。  関連付けられる候補は、関連付けが意味を持つエクスポートで、同じタイプの別のサブオブジェクトとの明示的な関連付けを既に持っていないものです。  後述するように、デフォルトの関連付けがエクスポートの既存の関連付けを上書きすることができる、1 つの例外がある。

---

#### Explicit associations

明示的関連付けは、与えられたサブオブジェクトを特定の非空白のエクスポートリストに関連付けます。

関連付けられるサブオブジェクト（例えば、ルート署名）および/またはリストされたエクスポートは、オブジェクト内の任意のスコープにすることができます。

さらに、状態オブジェクトが実行可能でない限り、関連付けられるサブオブジェクトもリストされたエクスポートもまだ可視である必要はない（未解決の参照であってもよい）（例えば、RTPSO など）。

---

#### Multiple associations of a subobject

与えられたサブオブジェクトは、複数の関連付け定義（明示的またはデフォルト）で参照することができます。この方法では、任意の関連付けの定義がすべてを知っている必要はありません（サブオブジェクトが関連する可能性があるすべてのシェーダを認識する必要はありません）。

複数のアソシエーション宣言の使用はまた、例えば、与えられたサブオブジェクトのためのデフォルトアソシエーションを複数のスコープにブロードキャストすることを可能にします。この場合、各関連付け宣言は（異なるスコープで）空のエクスポートを使用しますが（それをデフォルトの関連付けとする）、同じサブオブジェクトを参照します。

---

#### Conflicting subobject associations

与えられたシェーダーエクスポートにマッピングされる複数の明示的なサブジェクトの関連付け（異なるサブオブジェクトの定義を持つ）がある場合、これはコンフリクトとなります。DXIL ライブラリの作成中に競合が発見された場合、ライブラリの作成は失敗します。その他、ステート・オブジェクトの作成中にコンフリクトが発見された場合、それは失敗します。

コンフリクトの判定では、与えられたステート・オブジェクト（または単一スコープの DXIL ライブラリ）内で、関連付け、関連付けされているサブオブジェクト、またはシェーダー・エクスポートを保持するスコープを気にすることはありません。これは、定義上、明示的な関連付けはステート オブジェクト（または DXIL ライブラリ）内のどこにでも到達できるためです。

1 つの例外として、競合が原因で失敗することはなく、代わりに優先順位が存在するため、オーバーライドが発生することがあります。

---

##### Exception: overriding DXIL library associations

直接含まれる DXIL ライブラリのエクスポートをターゲットとするステート・オブジェクトで直接宣言されたサブオブジェクト関連付け（デフォルトまたは明示的）は、任意の DXIL ライブラリ（同じまたは他のライブラリ）で定義されたそのエクスポートへの他の関連付けを、そのエクスポートに適用しなくなるようにする。そしてその結果、ステート・オブジェクトで直接宣言されたサブオブジェクトの関連付けが「勝ち」、DXIL ベースの関連付けを上書きする。これには、D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION（同じくステートオブジェクトスコープで定義されたサブオブジェクトに対するステートオブジェクトスコープでの関連付け）または D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION（DXIL ライブラリで定義されたオブジェクトに対するステートオブジェクトスコアでの関連付け）を通じてステートオブジェクトスコアで定義された関連付けが含まれます。

ステートオブジェクトで宣言された関連付けは、含まれるコレクション（含まれるコレクションが持つ可能性のある DXIL ライブラリを含む）の既存の関連付けを上書きしません。

ステートオブジェクトが RTPSO（実行可能）でない限り、関連付けられるサブオブジェクトは未解決である可能性がある。

> The reason overriding is only defined for DXIL libraries directly passed
> into a given state object's creation is the following. Drivers never
> have to worry about compiling code that came from a DXIL library during
> state object creation only to have to recompile later because multiple
> subobject overrides happened. e.g. creating a collection that overrides
> associations in a DXIL library then creating an RTPSO that includes the
> collection and tries to override an association again is invalid (the
> second association becomes conflicting and state object creation
> fails).
>
> The value in supporting overriding of subobject associations is to give
> programmatic code (i.e. performing state object creation) one chance to
> override what is in a static DXIL library, without having to patch the
> DXIL library itself.

---

#### Subobject associations for hit groups

[Hit groups](#hit-groups) reference a set of component shaders, such as
a closest hit shader, any hit shader, and/or intersection shader.
Subobject associations (like associating a local root signature to a
shader) can be made directly to the individual component shaders used by
a hit group and/or directly to the hit group. Making the association to
the hit group can be convenient, as it applies to all the component
shaders (so they don't need individual associations). If both a hit
group has an association and its component shaders have associations,
they must match. If a hit group doesn't have a particular subobject
association, the associations for all component shaders must match. So
different component shaders can't use different local root signatures,
for instance.

---

#### Runtime resolves associations for driver

ランタイムは、デフォルトやオーバーライドなどを考慮して、任意のエクスポートで終了したサブオブジェクトの関連付けを解決し、ステートオブジェクトの関連付けの間にその結果をドライバーに伝えます。これにより、実装間の一貫性が保証される。

---

### Subobject association requirements

この表は、シェーダーエクスポートとの関連付けをサポートする各サブオブジェクトタイプの要件を記述しています。

Match rule" 列には、任意のシェーダに対してサブオブジェクトの関連付けが必須か任意か、またサブオブジェクトの定義が他のシェーダの定義と一致しなければならないかどうかが記述されています。

Match scope" 列には、マッチング要件がシェーダコードのどのセットに適用されるかが記述されています。

Subobject typeMatch ruleMatch scope[Raytracing shader config](#d3d12_raytracing_shader_config)Required & matching for all exportsFull state object. More discussion at [D3D12_RAYTRACING_SHADER_CONFIG](#d3d12_raytracing_shader_config).[Raytracing pipeline config](#d3d12_raytracing_pipeline_config1)Required & matching for all exportsFull state object. More discussion at [D3D12_RAYTRACING_PIPELINE_CONFIG](#d3d12_raytracing_pipeline_config), [D3D12_RAYTRACING_PIPELINE_CONFIG1](#d3d12_raytracing_pipeline_config1).[Global root signature](#d3d12_global_root_signature)Optional, if present must match shader entryCall graph reachable from shader entry. More discussion at [D3D12_GLOBAL_ROOT_SIGNATURE](#d3d12_global_root_signature).[Local root signature](#d3d12_local_root_signature)Optional, if present must match shader entryCall graph reachable from shader entry (not including calls through shader tables). More discussion at [D3D12_LOCAL_ROOT_SIGNATURE](#d3d12_local_root_signature).[Node mask](#d3d12_node_mask)Optional, if present match for all exportsFull state object. More discussion at [D3D12_NODE_MASK](#d3d12_node_mask).[State object config](#d3d12_state_object_config)Optional, if present match for all exportsLocal state object only, doesn't need to match contained state objects. More discussion at [D3D12_STATE_OBJECT_CONFIG](#d3d12_state_object_config)---

### State object caching

ドライバは、D3D12 の既存のサービスを使用してステートオブジェクトのキャッシングを実装し、ステートオブジェクト（またはその中のコンポーネント）がアプリケーションの実行間で再利用されるときのパフォーマンスを向上させる責任を負います。

---

### Incremental additions to existing state objects

[Tier 1.1](#d3d12_raytracing_tier) implementations support adding to existing state objects via
[AddToStateObject()](#addtostateobject). This incurs lower CPU overhead in streaming scenarios where new shaders need to be added to a state object that is already being used, rather than having to create a state object that is mostly redundant with an existing one. Details on the nuances of this option are described at [AddToStateObject()](#addtostateobject).

---

# System limits and fixed function behaviors

---

## Addressing calculations within shader tables

> The very fixed nature of shader table indexing described here is a
> result of IHV limitation. The hope is these limitations aren't too
> annoying for apps (which have to live with them). The extent to which
> the fixed function choices made here conflict with what an app actually
> wants may force app to do inefficient things like duplicating entries in
> shader tables to accomplish what they want. That said, such
> inefficiencies in shader table layout may not turn out to be an overall
> bottleneck. So this might be no worse than simply being slightly awkward
> to use.

---

### Hit group table indexing

> HitGroupRecordAddress =
> [D3D12_DISPATCH_RAYS_DESC](#d3d12_dispatch_rays_desc).HitGroupTable.StartAddress \+ <small>// from: [DispatchRays()](#dispatchrays)</small><br> > [D3D12_DISPATCH_RAYS_DESC](#d3d12_dispatch_rays_desc).HitGroupTable.StrideInBytes \* <small>// from: [DispatchRays()](#dispatchrays)</small><br>
> (
> RayContributionToHitGroupIndex \+ <small>// from shader: [TraceRay()](#traceray)</small><br>
> (MultiplierForGeometryContributionToHitGroupIndex \* <small>// from shader: [TraceRay()](#traceray)</small><br>
> GeometryContributionToHitGroupIndex) \+ <small>// system generated index of geometry in bottom-level acceleration structure (0,1,2,3..)</small><br> > [D3D12_RAYTRACING_INSTANCE_DESC](#d3d12_raytracing_instance_desc).InstanceContributionToHitGroupIndex <small>// from instance</small><br>
> )

MultiplierForGeometryContributionToHitGroupIndex > 1 を設定すると、アプリケーションはシェーダテーブル内でジオメトリごとに隣接する複数のレイタイプ用のシェーダをグループ化することができま す。アクセラレーション構造は、単にインスタンスごとの InstanceContributionToHitGroupIndex を格納するだけであるため、これが起こっていることを知る必要はありま せん。GeometryContributionToHitGroupIndex は、固定関数の順次インデックス（0、1、2、3...）で、ジオメトリごとに増加し、各ジオメトリが現在のボトムレベルのアクセラレーション構造でアプリによって置かれた順序をミラーリングしています。

レイトレーシング Tier 1.1 の実装では、MultiplierForGeometryContributionToHitGroupIndex を 0 に設定して、ジオメトリインデックスがシェーダテーブルイン デキシングに全く寄与しないようにすると面白いことがあります。 これは、シェーダが GeometryIndex() 組込み関数 (Tier 1.1 の実装で追加) を呼び出して、シェーダ内のジオメトリを手動で区別できるようにする場合に有効で す。

---

### Miss shader table indexing

> MissShaderRecordAddress =
> D3D12_DISPATCH_RAYS_DESC.MissShaderTable.StartAddress \+ <small>// from: [DispatchRays()](#dispatchrays)</small><br>
> D3D12_DISPATCH_RAYS_DESC.MissShaderTable.StrideInBytes \* <small>// from: [DispatchRays()](#dispatchrays)</small><br>
> MissShaderIndex <small>// from shader: [TraceRay()](#traceray)</small><br>

---

### Callable shader table indexing

> CallableShaderRecordAddress =
> D3D12_DISPATCH_RAYS_DESC.CallableShaderTable.StartAddress + <small>// from: [DispatchRays()](#dispatchrays)</small><br>
> D3D12_DISPATCH_RAYS_DESC.CallableShaderTable.StrideInBytes \* <small>// from: [DispatchRays()](#dispatchrays)</small><br>
> ShaderIndex <small>// from shader: [CallShader()](#callshader)</small>

---

### Out of bounds shader table indexing

シェーダーテーブルが範囲外のインデックスを持つ場合の動作は未定義です。初期化されていない、または古いデータを含むシェーダーテーブル内のリージョンを参照する場合も同様です。

---

## Acceleration structure properties

---

### Data rules

- アクセラレーション構造は、トップレベルのアクセラレーション構造がボトムレベルのアクセラレーション構造を指すことを除けば、自己完結しています。

- アプリケーションは、アクセラレーション構造体の内容を検査することはできません。しかし、このデータは実装に依存し、文書化されていないため、アプリが検査することはできません。

- 一度構築されたアクセラレーション構造は、インプレースで行われる更新（インクリメンタルビルド）を除いて、不変です。

- トップレベルのアクセラレーション構造は、それが参照するボトムレベルのアクセラレーション構造がリビルドまたは更新されるたびに、使用前にリビルドまたは更新する必要があります。

- アクセラレーション構造体に対する有効な操作は次のとおりです。

- BuildRaytracingAccelerationStructure() への入力。

  - input to [TraceRay()](#traceray) and [RayQuery::TraceRayInline()](#rayquery-tracerayinline) from a shader

  - トップレベルのアクセラレーション構造体のビルドによって参照されるボトムレベルの構造体として。

    - acceleration structure の更新（インクリメンタルビルド）のソースとして。

    - ソースはインプレース更新を意味するため、デスティネーションアドレスと同じにすることができます。

      - CopyRaytracingAccelerationStructure() への入力で、acceleration structure の圧縮やデータ構造の単純な複製など、さまざまなモードがあります。

    - 特に、他の方法でアクセラレーション構造をコピーすることは無効であることに注意してください。

      - EmitRaytracingAccelerationStructurePostbuildInfo() への入力で、圧縮されたバージョンにどれだけのスペースが必要かといった acceleration structure に関する情報を報告します。

    - 頂点の順序（三角形の場合）

---

### Determinism based on fixed acceleration structure build input

Given a fixed world composed of triangles and AABBs, as well as
identical shader code and data in the same order, multiple identical
[TraceRay()](#traceray) or [RayQuery::TraceRayInline()](#rayquery-tracerayinline) calls produce identical results on the same
device and driver. This requirement means that both the tracing of rays
must be deterministic, and the acceleration structure must also be
constructed such that it behaves deterministically.

同じ三角形ストリーム、AABB ストリーム、および複数のアクセラレーション構造体構築へのその他の設定入力（該当する場合は同じインスタンスとジオメトリ変換およびその他のプロパティを含む）がある場合、結果として得られるアクセラレーション構造体の動作は所定のデバイスとドライバーで同じでなければなりません。実際の acceleration structure の内容はビット単位で同一ではない可能性があり、これはメモリ比較で明らかになる可能性があります。例えば、異なるアドレスやデータレイアウトの順序を参照する内部ポインターを含んでいても、動作に影響を与えることはありません。つまり、一致しなければならないのは、一貫して構築された acceleration structure の機能的な動作だけなのだ。アプリケーションシェーダと実行フローに影響するデータも一致すれば、シェーダ起動の順番が同じであれば、同じ交差点が見つかります。

アクセラレーション構造の更新（インクリメンタルビルド）では、各更新の入力セットが一致する複数の同一の更新シーケンスは、上記のアクセラレーション構造の動作と同じ一貫性をもたらします。

---

### Determinism based varying acceleration structure build input

アクセラレーションを構築するために使用するジオメトリの位置と量を変更すると、その動作に影響を与えるという明白な事実の他に、アクセラレーション構造の機能に影響を与える可能性のある微妙なバリエーションが存在します。

acceleration structure の交差点検出と交差点順序付けの動作は、acceleration structure の構築における以下の要因のいずれかが変化した結果、変化する可能性があります。

- プリミティブオーダー（三角形の場合）

- AABB 順序

- トップレベルの acceleration structure におけるインスタンス順序

- Bottom-Level Acceleration Structure におけるジオメトリ順序

- acceleration structure 構築のフラグ（またはインスタンス/ジオメトリフラグ）

- acceleration structure 更新（インクリメンタルビルド）回数と入力履歴

- デバイス/ドライバ

- シェーダーテーブルのインデックス計算またはシェーダー ID に寄与するアクセラレーション構造体に埋め込まれたユーザー定義値。実装では、たとえば、これらの値でコンテンツをソートしたり、同じ値を使用するコンテンツのセットを何らかの方法で知る理由が見つかるかもしれません。もちろん、acceleration structure の構築中に実際のシェーダーテーブルは存在しないので、実装が見ることができるのは、それらを使用しようとせずに生のオフセット/ ID 値だけです。

- アクセラレーション構造またはビルド入力のメモリアドレス（前述のデータ順序の公差は別として）。

acceleration structure の交差点検出と交差点順序付けの動作は、acceleration structure のビルド間で以下のいずれの要因も変化させませ ん。

- 時間

- インターセクションシェーダの呼び出し回数。

---

### Preservation of triangle set

実装は、頂点の順序とプリミティブの順序を除いて、アクセラレーション構造内の三角形の入力セットを変更することはできません。トライアングルのマージ、スプリット、ドロップは許されない。

アクセラレーション構造におけるプリミティブの観測可能な重複は無効とする。観測可能とは、性能差だけでなく、レイトレーシング動作中に見えるようになるものであれば何でも良い。例外は以下の通り。

- アプリケーションが与えられたジオメトリで `D3D12_RAYTRACING_GEOMETRY_FLAG_NO_DUPLICATE_ANYHIT_INVOCATION` を設定していない場合、与えられたプリミティブで与えられたレイの複数の any hit invocation が観察されることがあります。

- シーンのエッジに当たる単一のレイは、入射する三角形のうちの 1 つだけとの交差を報告しなければなりません。頂点に当たったレイは、入射する三角形のうちの 1 つとの交点を報告しなければなりません。どの三角形が選択されるかは、同じエッジに交差する異なるレイで異なる場合があります。

三角形交差の交差点属性構造で提供されるバリセントリックは、アプリがそれ自身で頂点属性を調べることができなければならないので、元の頂点の順序に対して相対的でなければなりません。

---

### AABB volume

実装では、acceleration structure 構築の入力として提供される AABB を、より多いまたは少ない AABB (または他の表現) に置き換えることができますが、入力 AABB で囲まれた空間内の位置が acceleration structure に含まれることのみが保証されます。

特に、アプリケーションは、acceleration structure 構築への入力 AABBs の平面に依存してはならず、囲まれた交差シェーダの呼び出しによって定義される形状に何らかのクリッピング効果を与えてはなりません。実装は交差点シェーダを呼び出すために、入力 AABB よりも大きなボリュームを選択することができます。この点については実装の自由ですが、バウンディングボリュームを過度に肥大化させると、不要な交差点シェーダの呼び出しによる極端な性能低下が発生します。したがって、バウンディングボリュームの肥大化の程度は、実際には制限されるべきです。

---

### Inactive primitives and instances

三角形は、いずれかの頂点の x 成分が NaN である場合、"inactive"（ただし、acceleration structure 構築のための正当な入力）とみなされる。同様に、AABB.MinX が NaN の場合、AABB は非アクティブとみなされる。bottom-level acceleration structure におけるジオメトリおよび／またはその変換は、これらの NaN 値を注入し、インスタンス／ジオメトリ全体を非アクティブにする 1 つの方法として使用することができる。NULL の Bottom-Level Acceleration Structure ポインターを持つインスタンスも、合法だが非活性であるとみなされます。

すべての非アクティブプリミティブ/AABBs/Bottom-Level Acceleration Structure は、アクセラレーション構造の構築段階で破棄され、したがってトラバーサル中に交差することはできません。

非アクティブなプリミティブは、その後の acceleration structure の更新でアクティブになることができません。逆に、最初の構築時にアクティブだったプリミティブは、acceleration structure の更新で非アクティブに変更することはできません（したがって、破棄することはできません）。

非アクティブプリミティブは PrimitiveIndex()でカウントされ、隣接するアクティブプリミティブのインデックス値に影響を与える。同様に、非アクティブなインスタンスは InstanceIndex() でカウントされ、隣接するインスタンスのインデックスに影響を与えます。これにより、アプリケーションがアクティブプリミティブのインデックス値に基づいて行いたい配列のインデックス付けが、配列の初期に存在する非アクティブプリミティブの影響を受けないことが保証されます。

三角形と AABB の場合、最初の座標が NaN でない限り（つまり、プリミティブが非アクティブの場合のみ）、入力座標はどれも NaN であってはならず、そうでない場合の動作は未定義である。

SNORM フォーマットを使用する三角形の頂点 (D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC の VertexFormat 参照) は、SNORM に NaN 表現がないため、inactive にはできません。

---

### Degenerate primitives and instances

以下のものは、（全ての変換を適用した後）「縮退している」と見なされます。点または線を形成する三角形、点サイズの AABB（Min.X == Max.X、Y と Z は同じ）、アクティブプリミティブを含まない（非 NULL）Bottom-Level Acceleration Structure を参照するインスタンス。

縮退したプリミティブとインスタンスはすべて「アクティブ」とみなされ、新しいデータによる acceleration structure の更新に参加でき、縮退状態との切り替えも自由に行えます。

縮退したインスタンスは、acceleration structure ビルダーのガイドとしてインスタンス原点（インスタンス変換によって指定される）のポイントを使用します。アプリは、将来の更新でインスタンスが配置される可能性のある場所の最良推定値を与える必要があります。

反転した境界（Min.X > Max.X または Y と Z の類似）を持つ AABB は、すべての acceleration structure 操作の入力境界の中心で縮退した点に変換されます。

トラバーサル中に、縮退した AABB はまだ可能性のある（偽陽性）交差を報告し、交差点シェーダを呼び出すことができる。シェーダは、例えば、境界を検査することによって、ヒットの妥当性をチェックすることができる。

一方、退化した三角形は、交差を生成しません。

退化したプリミティブは、 `ALLOW_UPDATE` フラグが指定されていない限り、acceleration structure の構築から廃棄される可能性があります。その結果、AS 可視化時に返される座標が NaN に置き換わってしまうことがある。

`ALLOW_UPDATE` を指定した場合、縮退は捨てられないというルールの例外として、インデックス値が繰り返されるプリミティブは（ `ALLOW_UPDATE` を指定していても）常に捨てられる。 インデックス値を変更することはできないので、それらを保持する価値はありません。

返される GeometryDescs のプリミティブ数と InstanceDescs の数は、破棄されたプリミティブの影響を受ける可能性があります。しかし、アクティブなプリミティブのインデックス（バッファの位置）と出力順序は、依然として正しいままです。

---

### Geometry limits

ランタイムはこれらの制限を強制しません(定義が遅すぎました)。この制限を超えると、未定義の動作が発生します。

Bottom-Level Acceleration Structure におけるジオメトリの最大数：2^24

ボトムレベル・アクセラレーション構造内の最大プリミティブ数(すべてのジオメトリの合計)。2^29 (非アクティブまたは縮退したプリミティブを含む)

トップレベルのアクセラレーション構造における最大インスタンス数：2^24

---

## Acceleration structure update constraints

以下は、ソース acceleration structure の構築に使用された入力/フラグなどに対して、acceleration structure 更新の入力にアプリが変更できるデータについて説明します。アクセラレーション構造体のデータ規則では、一度構築されると、構築のために使用されたデータへの明示的な参照を保持しないので、データ自体の変更のみが以下の制限に準拠している限り、更新がメモリ内の異なるアドレスからデータを提供しても問題ないことに注意してください。

> A rule of thumb is that the more that acceleration structure updates
> diverge from the original, the more that raytrace performance is likely
> to suffer. An implementation is expected to be able to retain whatever
> topology it might have in an acceleration structure during update.

---

### Bottom-level acceleration structure updates

D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC の VertexBuffer と Transform メンバーを変更することができます。ただし、Transform メンバは NULL <-> non-NULL の間で変更することはできません。Transform を更新したいが初期状態では持っていないアプリは、NULL ではなく ID 行列を指定することができます。

基本的に、これは頂点の位置が変更できることを意味します。

D3D12_RAYTRACING_GEOMETRY_DESC の AABBs メンバは変更することができます。

それ以外のものは変更できませんので、特に、ジオメトリ数、VertexCount、AABB などのプロパティ、ジオメトリフラグ、データフォーマット、インデックスバッファの内容などが変更されないことに注意してください。

あるアドレスにある Bottom-Level Acceleration Structure が Top-Level Acceleration Structure によって指される場合、それらの Top-Level Acceleration Structure は古くなり、再び使用する前に再構築または更新する必要があることに注意してください。

---

### Top-level acceleration structure updates

D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC の InstanceDescs メンバは変更することができます。

これは GPU メモリ内の D3D12_RAYTRACING_INSTANCE_DESC 構造体を参照します。D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC の NumDescs メンバーで定義されるインスタンスの数は、変更することができ ません。

したがって、トップレベルのアクセラレーション構造で使用されるインスタンスの数は固定されていますが、各インスタンスの定義はアクセラレーション構造の更新中に完全に再定義することができ、各インスタンスがどのボトムレベルのアクセラレーション構造を指すかも含みます。

---

## Acceleration structure memory restrictions

アクセラレーション構造は、デフォルトヒープ（またはカスタムヒープ相当）で作成されたリソースにのみ配置することができます。さらに、アクセラレーション構造を含むリソースは D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE 状態で作成され、リソースフラグ `D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS` を持っていなければな りません。 `ALLOW_UNORDERED_ACCESS` 要件は、システムがアクセラレーション構造体の構築の実装でこのタイプのアクセスを舞台裏で行うことと、アプリの観点から、アクセラレーション構造体への書き込み/読み込みの同期が UAV バリア（後述）で実現することの両方を単に認識するものです。

以下の説明では、これらのリソースを加速構造バッファ（ASB）と呼ぶ。

ASB は他の状態に遷移することはできず、またその逆もできない。さもなければ、ランタイムはコマンドリストを削除された状態に置くことになる。

ASB である配置バッファを作成したが、ASB でない VA 範囲に重なる既存のバッファがある場合、またはその逆の場合、これはデバッグレイヤーエラーによって強制されるエラーである。

予約バッファについて、あるタイルが ASB と非 ASB に同時にマップされた場合、これはデバッグレイヤーエラーによって強制されるエラーとなります。acceleration structure へのタイルのマッピングまたは acceleration structure からのタイルは、そのタイルの内容を無効にします。

> The reason for segregating ASBs from non-ASB memory is to enable
> tools/PIX to be able to robustly capture applications that use
> raytracing. The restriction avoids instability/crashing from tools
> attempting to serialize what they think are opaque acceleration
> structures that might have been partially overwritten by other data
> because the app repurposed the memory without tools being able to track
> it. The key issue here is the opaqueness of acceleration structure data,
> requiring dedicated APIs for serializing and deserializing their data to
> be able to preserve application state.

---

### Synchronizing acceleration structure memory writes/reads

アクセラレーション構造は常に D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE でなければならないと考えると、リソース状態遷移はアクセラレーション構造データの書き込みと読み込み（またはその逆）の間の同期に使用することはできま せん。代わりに、acceleration structure への書き込み操作（BuildRaytracingAccelerationStructure()など）と読み込み操作（DispatchRays()など）（およびその逆）の間で、acceleration structure データを保持するリソースに UAV バリアを使用してこれを達成する方法がある。

> The use of UAV barriers (as opposed to state transitions) for
> synchronizing acceleration structure accesses comes in handy for
> scenarios like [compacting](#d3d12_raytracing_acceleration_structure_copy_mode) multiple acceleration structures.
> Each compaction reads an acceleration structure and then writes the
> compacted result to another address. An app can perform a string of
> compactions to tightly pack a collection of acceleration structures that
> all may be in the same resource. No resource transitions are necessary.
> Instead all that's needed are a single UAV barrier after one or more
> original acceleration structure builds are complete before passing them
> into a sequence of compactions for each acceleration structure. Then
> another UAV barrier after compactions are done and the acceleration
> structures are referenced by DispatchRays() for raytracing.

---

## Fixed function ray-triangle intersection specification

多様体ジオメトリの場合。

- 2 つ以上の三角形が共有するエッジに光線が当たった場合、エッジの片側にあるすべての三角形 (光線の視点から) と交差します。光線が別々のサーフェスで共有される頂点に当たった場合、サーフェスごとに 1 つの三角形が交差します。光線が別々のサーフェスが共有する頂点に当たり、同じ場所でエッジに当たる場合、点とエッジの個別のルールに基づいて、点の交差点とエッジの交差点がそれぞれ表示されます。

![sharedEdge](images/raytracing/sharedEdge.png)![sharedVertex](images/raytracing/sharedVertex.png)

上記の共有エッジ交差点と共有頂点交差点の例では、それぞれのケースで 1 つの三角形だけが交差したものとして報告されなければなりません。

非多様体ジオメトリの場合。

- 各光線について、光線と三角形の交差が行われる平面を選択します。当然ながら、この平面はレイと交差する必要があり、レイを含まない場合もあります。この平面は、レイ自体(原点と方向)の関数でしかなく、他のレイが選んだ同じ平面でない可能性もあります。

---

### Watertightness

実装では、水密性のある光線-三角形交差を使用しなければなりません。32-bit float 精度の範囲内でレイトライアングル交差が発生する場所に関係なく、エッジを共有するトライアングル間のギャップは決して現れてはいけません。この水密性の定義における「共有」の範囲は、変換が一致するジオメトリの特定の Bottom-Level Acceleration Structure のみに及びます。

水密性のある光線-三角形交差の実装の一例はこちらです。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<http://jcgt.org/published/0002/01/05/paper.pdf>

以下は、水密性を維持しながら acceleration structure の実装を効率的に行うことに焦点を当てた別の例です。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<https://software.intel.com/en-us/articles/watertight-ray-traversal-with-reduced-precision>

倍精度フォールバックなどのコストのかかる方法に頼らずに、水密なレイトライアングル交差の実装が可能であることが期待されます。これには、エッジのダブルヒットを除去するために後述する左上ルールに従うことが含まれます。

---

#### Top-left rule

三角形ラスタライゼーションでは、左上のルールは、共有エッジに穴やダブルヒットがなく、実装間で三角形エッジの一貫したイン/アウト判定を保証しています。これは、一貫した空間（"スクリーン空間"）が存在し、ラスタライズ中に固定点精度への頂点位置のスナップが行われるため、ばらつきがないため可能なのです。

これらの条件は、レイトレーシングには存在しません。したがって、左上のルール（または同等）のいくつかの形式は、エッジ上の交差が in か out かを決定するためにレイトライアングル交差を行う各実装によって使用されなければならないが、これはその実装のための水密性を保証するだけで、実装間で正確に同じ結果ではありません。

以下は、実装によって適用可能な左上のルールの例です。この要件は、単に実装がこのようなことを行って、前述の交差特性を保証することです。

---

### Example top-left rule implementation

---

#### Determining a coordinate system

- レイと平面の交点を計算します。

- 交点を原点として、平面内で方向を選択し、これもレイだけの関数とします。この方向を左方向とします。

- 左方向と平面法線の積をとって、上方向とします。これで、左上の法則を実現するのに必要な座標系が出来上がりました。

- 三角形をレイの平面に投影します。

![determiningCoordinates](images/raytracing/determiningCoordinates.png)

---

#### Hypothetical scheme for establishing plane for ray-tri intersection

支配的な光線方向を使用することは、交差を実行する平面を確立する 1 つの方法です。これは、3 つの主要な軸に一致する 3 つの平面の方向から 1 つを選択することになります。交差プレーンのオフセットは重要ではありません。

以下のイメージは、光線方向と交差平面の対応付けの例です。単位球面上の光線方向は、3 つの可能な交差平面にマッピングされます。

![directionMapping1](images/raytracing/directionMapping1.png)![directionapping2](images/raytracing/directionMapping2.png)

法線の次に大きな大きさの成分に基づいて、平面内の左方向を選択します。

下の画像では、これは色のついた三角形の領域と、それに対応する平面内の左方向に対応します。3 つの可能な平面には、それぞれ 2 つの可能な左方向があります。

![leftDirection](images/raytracing/leftDirection.png)

---

#### Triangle intersection

- 三角形を光線と平面の交点に対してテストします。光線と平面の交点が三角形の厳密に内側にある場合、交点を報告する。

- 光線と平面の交点が投影された三角形の辺の一つに直接ある場合、左上のルールを適用して、三角形が交差しているとみなされるかどうかを確定する。

- 光線と三角形の交点は、ハードウェアで加速することができます。

  **\*Top edge**: If a projected edge is exactly parallel to the left
  direction, and the* up *direction points away from the projected
  triangle's interior in the space of the ray's plane, then it is a
  "top" edge.\*

  **\*Left edge**: If a projected edge is not exactly parallel to the
  left direction, and the* left *direction points away from the
  projected triangle's interior in the space of the ray's plane, then it
  is a "left" edge. A triangle can have one or two left edges.\*

  **\*Top-left rule**: If the ray-plane intersection falls exactly on the
  edge of a projected triangle, the triangle is considered intersected
  if the edge is a "top" edge or a "left" edge. If two edges from the
  same projected triangle (a vertex) coincide with the ray-plane
  intersection, then if both edges are "top" or "left" then the triangle
  is considered intersected.\*

---

#### Examples of classifying triangle edges

下の画像では、交差平面内で 1 つの左方向（赤）が示されている。包含的なエッジは黒で、排他的なエッジは白で表示されている。青とマゼンタの三角形は、左方向と平行な辺を持っていることに注意。一方は上側のエッジ、他方は下側のエッジで、それぞれ包括的、排他的である。

![classifyingEdges](images/raytracing/classifyingEdges.png)

一連の光線がエッジに当たると、左方向が 90° 変化することがある。左方向が変化するため、エッジの長さに沿ってエッジの包含/排他分類が変化することがある。また、エッジの長さ方向に交差面が変化した場合（これも光線方向の関数）にも分類が変化することがあります。

---

## Ray extents

レイトライアングル交差は、交差点の t 値が TMin < t < TMax を満たす場合にのみ発生します。

レイプロシージャープリミティブ交差は、交差点 t-value が TMin <= t <= TMax を満たす場合にのみ発生することができます。

> The reason procedural primitives use an inclusive bounds test is to
> give apps a choice about how to handle exactly overlapping intersections.
> For instance, an app could choose to compare primitiveID or somesuch for
> currently committed hit versus a candidate for intersection to decide whether
> to accept a new overlapping hit or not.

レイ TMin は非負であり、かつ ≦TMax でなければならない。+INF は有効な TMin/TMax 値です(実際には TMax に対してのみ意味を持ちます)。

レイの原点、方向、T 範囲のどの部分も NaN にすることはできません。

ランタイムはこれらの制限を強制しません(GPU ベースの検証でいずれ検証されるかもしれません)。

これらの規則に違反した場合、未定義の動作が発生します。

---

## Ray recursion limit

Raytracing pipeline state objects must [declare](#d3d12_raytracing_pipeline_config) a maximum ray recursion
depth (in the range [[0..31]](#constants)). The ray generation shader
is depth 0. Below the maximum recursion depth, shader invocations such
as closest hit or miss shaders can call [TraceRay()](#traceray) any
number of times. At the maximum recursion depth, [TraceRay()](#traceray)
calls result in the device going into removed state.

現在の再帰レベルは、シェーダに報告できるようにするために必要なオーバーヘッドのために、システムから取得することができません。アプリケーションがレイ再帰のレベルを追跡する必要がある場合、レイペイロードで手動で行うことができます。

この recursion depth の制限には、callable シェーダは含まれず、パイプラインスタッ ク割り当て全体のコンテキストを除いて、制限を受けません。

> Apps should pick a limit that is as low as absolutely necessary. There
> may be performance implications in how the implementation chooses to
> handle upper limits set at obvious thresholds -- e.g. 0 means no tracing
> of rays at all (perhaps only using callable shaders or not even that), 1
> means single bounce rays, and numbers above 1 might imply a different
> implementation strategy.
>
> It isn't expected that most apps would ever need to declare very large
> recursion limits. The upper limit of 31 is there to put a bound on the
> number of bits hardware has to reserve for a counter -- inexpensive yet
> large enough range to likely never have to worry about.

---

## Pipeline stack

呼び出し可能なシェーダを含むレイトレーシングのシェーダは、ドライバが管理するスタック割り当てのうち、メモリを消費することがあります。このメモリはコマンドリストの記録中にドライバによって内部的に割り当てられ、予約されます。このため、選択されたスタックサイズのためにドライバがコマンドリストを実行できない場合、コマンドリストの記録は失敗します。スタックメモリの要件は、個々のレイ生成シェーダースレッドから始まるコールチェーンが、トレース光線、呼び出し可能なシェーダを含むプロセスで呼び出すことができる様々なシェーダ、およびネスト/再帰を考慮して、どのくらいのメモリを消費することができるかという観点から表現されています。

> In practice typical systems will support many thousands of threads in
> flight at once, so the actual memory footprint for driver managed stack
> storage will be much larger than the space required for just one thread.
> This multiplication factor is an implementation detail not directly
> exposed to the app. That said, it is in the app's best interest (if
> memory footprint is important) to make an optimal choice for the one
> number it has control over -- individual thread stack size -- to match
> what the app actually needs.

レイトレーシング パイプライン ステート オブジェクトは、オプションで最大パイプライン スタック サイズを設定できますが、そうでない場合はデフォルト値が使用され、これは通常過度に保守的です（ただし、呼び出し可能なシェーダが 2 レベル以上再帰する場合は過小評価される可能性があります）。次に、アプリが最適なスタックサイズを計算する方法を説明し、その後、デフォルトがどのように選択されるかを説明します。

> Here is an example of a situation where it really matters for an app to
> manually calculate the stack size rather than rely on the default:
> Suppose there is a complex closest hit shader with lots of state doing
> complex shading that recursively shoots a shadow ray that's known to hit
> only trivial shaders with very small stack requirements. The default
> calculation described further below doesn't know this and will assume
> all levels of recursion might invoke the expensive closest hit shader,
> resulting in wasted stack space reservation, multiplied by the number of
> threads in flight on the GPU.

---

### Optimal pipeline stack size calculation

アプリは GetShaderStackSize()を介してレイトレーシング パイプラインの個々のシェーダのスタックスペース要件を取得することができま す。(その結果は、あるシェーダが他のレイトレー シングパイプラインにある場合でも同じになります)。アプリがこれらのサイズをレイトレーシング中の個々のシェーダ間の最悪ケースのコールスタックについて知っているかもしれないことと、それが宣言した MaxTraceRecursionDepth とともに組み合わせれば、正しいスタックサイズを計算することができま す。これは、システムが自分ではできないことです。

下の図は、レイトレーシングのパイプラインで使用されているシェーダと、どのシェーダが到達可能か（これはアプリの作者だけが合理的に知ることができます）に基づいて、アプリの作者が最適なスタックサイズを計算することを推論する方法を示しています。

![shaderCallStackConstruction](images/raytracing/shaderCallStackConstruction.png)

アプリは SetPipelineStackSize() を介してレイトレーシング パイプライン・ステートのスレッドごとの全体的なスタック ストレージを設定することができます。そのメソッドの仕様は、パイプライン・ステートに対していつ、どのくらいの頻度でスタックサイズを設定できるかについての規則を記述しています。

---

### Default pipeline stack size

システムはレイトレーシングのパイプライン・ステート・オブジェクトを、次のように計算されたデフォルトのパイプラインスタックサイズで初期化します。この計算は意図的に単純化されており、実際に実行されるかもしれないシェーダの組み合わせを考慮することはできません（アプリケーションの内容とシェーダテーブルのレイアウトに依存するため、レイトレーシング パイプライン・ステートの観点からはどちらも未知です）。デフォルトのスタックサイズ計算では、個々のスタックサイズの点で、レイトレー シングパイプラインのシェーダの最悪の組み合わせと、最大再帰レベルに係数を取 ります。呼び出し可能なシェーダの場合、デフォルトの仮定は、すべてのレイトレー シングシェーダが最大スタックサイズで呼び出し可能なシェーダを再帰深 度 2 で呼び出すというものです。

その結果、呼び出し可能なシェーダを持たないレイトレー シングパイプラインでは、デフォルトのスタックサイズは、再帰の最大宣言 レベルとシェーダの最悪のケースの組み合わせに安全に適合することが保証され ます。呼び出し可能なシェーダが混在している場合、アプリが他のすべてのシェーダを最大にした上で、最悪のケースの呼び出し可能なシェーダに 2 レベル以上の再帰呼び出しをすることが起こると、デフォルトは安全でない可能性があります。

正確な計算方法は次のとおりです。まず、入力変数の定義です。

パイプライン・ステートにある各シェーダタイプについて、そのシェーダタイプの複数のインスタンスがパイプライン・ステートにある可能性がある場合、そのシェーダタイプの個々の最大シェーダスタックサイズを求めます。この議論では、これらの最大値に名前を付けることにしましょう。

**RGSMax** (for max ray generation shader stack size)
**ISMax** (intersection shader)
**AHSMax** (any hit shader)
**CHSMax** (closest hit shader)
**MSMax** (miss shader)
**CSMax** (callable shader)

他の関連する入力は、パイプライン・ステートの D3D12_RAYTRACING_PIPELINE_CONFIG サブオブジェクトから来るものです。MaxTraceRecursionDepth。

これらの値を使用して、上記のレイトレーシングのシェーダコールスタック構築図を考慮し、呼び出し可能なシェーダがシェーダステージごとに 2 レベルの深さで呼び出されるように任意に推定すると、デフォルトスタックサイズの計算は次のようになります。

```C++
DefaultPipelineStackSizeInBytes =
  RGSMax
  + max( CHSMax, MSMax, ISMax+AHMax ) * min( 1, MaxTraceRecursionDepth )
  + max( CHSMax, MSMax ) * max( MaxTraceRecursionDepth - 1, 0 )
  + 2 * CSMax // if CS aren't used, this term will just be 0.
              // 2 is a completely arbitrary choice

  // Observe that ISMax and AHMax are only counted once, which results in
  // the split clauses involving MaxTraceRecursionDepth. Intersection and
  // anyhit shaders can't recurse.
```

---

### Pipeline stack limit behavior

呼び出しが宣言されたスタックサイズを超えると、レイ再帰のオーバーフローと同様に、デバイスは削除状態になります。

宣言されたスタックサイズに実用的な制限はない。ランタイムは、極端なスタックサイズ値（>= 0xffffffff）に対する SetPipelineStackSize() の呼び出しをドロップします（この目的のために、パラメータは実際には UINT64 です）。これは、アプリが無効なパラメータで GetShaderStackSize() を呼び出して 0xffffff を返す戻り値を、直接 SetPipelineStackSize に渡すか、スタックサイズを合計する計算 に渡して、そのうちの複数が無効な値である可能性があることを、やみくもに捕捉するため です。

---

## Shader limitations resulting from independence

レイトレーシングのシェーダの呼び出しはすべて互いに独立しているため、シェーダ内の機能で、シェーダ間の通信に明示的に依存するものは、以下で説明する Wave Intrinsics を除き、許可されていません。レイトレーシング中にシェーダが利用できない機能の例：2x2 シェーダ呼び出しに基づく微分（ピクセルシェーダで利用可能）、スレッド実行同期（コンピューティングで利用可能）。

---

### Wave Intrinsics

Wave intrinsics はツール（PIX）ロギング用であることを意図して、レイトレー シングシェーダで許可されています。つまり、アプリケーションは、安全な使用を見つけるかもしれない場合に備えて、wave intrinsics を使用することからブロックされてはいません。

実装は、TraceRay()への呼び出しのようなレイトレーシング シェーダ実行の特定の（よく定義された）ポイントでスレッドを再パックするかもしれ ません。そのため、シェーダ内で呼び出された wave intrinsics の結果は、プログラムの実行順序で潜在的なスレッドリパッキングポイントに遭遇するまでの間のみ有効である。ウェーブ・イントリシックスには、シェーダーの開始/終了点とリパッキング・ポイントに 境界された有効スコープがあります。

wave 固有のスコープを束縛するリパッキングポイント。

- [CallShader()](#callshader)

- [TraceRay()](#traceray)

- [ReportHit()](#reporthit)

シェーダー呼び出しの終了によりバインドされる他のイントリンシックス。

- [IgnoreHit()](#ignorehit)

- [AcceptHitAndEndSearch()](#accepthitandendsearch)

インラインレイシングのための RayQuery オブジェクトの使用は、リパッキングポイントとし てカウントされないことに注意してください。

---

## Execution and memory ordering

TraceRay()または CallShader()が呼び出されたとき、結果として生じるすべてのシェーダの呼び出しは、呼び出しが戻るまでに完了します。

呼び出し側で実行されたメモリ操作（ストア、アトミック）は、呼び出し側または呼び出し側のどちらかにメモリバリアがあれば、呼び出し側から見えることが保証されます。 `DeviceMemoryBarrier()` は、レイトレーシング シェーダに関連するバリアコールです。入力ペイロード/パラメータデータは値で callee に渡されるため、バリアは必要ありません。

呼び出し側によって実行されるメモリ操作は、同様に、任意のシェーダでメモリバリアを使用して呼び出し側（および呼び出し側によって行われる後続のすべての呼び出し）から見えることが保証されることができます。 出力ペイロード/パラメータデータは値で呼び出し元に返されるため、バリアは必要ありません。

# General tips for building acceleration structures

以下の一般的なアドバイスは BuildRaytracingAccelerationStructure() の使用 に適用されるものです。時間の経過とともに、より多様なデバイスのサポートが現れると、アドバイスを改良する必要がある可能性がありますが、そのままでも、これは遊びで様々なオプションの有用なリマインダーであるべきです。

- **Prefer triangle geometry over procedural primitives**

  - ジオメトリが実行するヒットシェーダコードを必要としない場合（アルファテスト用など）、レイトレーシングのハードウェアをできるだけ効果的に利用するために、常に OPAQUE としてマークされていることを確認します。OPAQUE フラグがジオメトリディスクリプタ（ `D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE` ）、インスタンスディスクリプタ（ `D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_OPAQUE` ）またはレイフラグ（ `RAY_FLAG_FORCE_OPAQUE` ）から来たかは問題ではないでしょう。

- **Mark geometry as OPAQUE whenever possible**

  - 言い換えれば、ビルドが複数のジオメトリ記述子を受け入れることができるという事実を利用し、ビルド中にジオメトリを変換します。これは一般に、特にオブジェクトの AABB が互いに重複している場合に、最も効率的なデータ構造につながります。さらに、BuildRaytracingAccelerationStructure の呼び出し回数が減るので、GPU の使用率が上がり、全体の CPU オーバーヘッドが減ります。たとえば、複数のメッシュからなるオブジェクト（および同時に再構築/更新する必要がある）、および静的またはほぼ静的なジオメトリのために、マージを検討してください。

- **Merge many objects into fewer bottom-level acceleration
  structures**

  - アクセラレーションの更新は無料ではないので、フレーム間で変形していないオブジェクトは更新をトリガーしない方がよいでしょう。エンジンでこれを検出するのは容易ではありませんが、スキニングやバーテックス更新のパスをスキップできる可能性があるため、この努力は 2 回報われることがあります。

- **Only build/update per frame what's really needed**

  - アクセラレーション構造の更新は、インプレースで行われる場合と、ソースと デスティネーションバッファを別々に使用する場合があります。  一部のジオメトリ（ヒーローのキャラクタなど）では、異なるキーポーズで複数の高品質なアクセラレーション構造を前もって構築しておき（レベルのロード時間中など）、最も近いキーフレームをソースとして使用してフレームごとに再フィットすることが理にかなっている場合があります。

- **Consider multiple update sources for skinned meshes**

  - 再構築の代わりに更新のみを行うことが、正しいことであることはほとんどありません。数千のインスタンスのリビルドは非常に高速で、質の良いトップレベルのアクセラレーション構造を持つことは、大きな見返りをもたらします（質の悪いものは、ツリーのさらに上の部分でより高いコストをもたらします）。

- **Rebuild top-level acceleration structure every frame**

  - 次のセクションは、一般的な使用例に対するガイドラインです。

- **Use the right build flags**

  - ここから D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS の組み合わせを選ぶところから始めます。

---

## Choosing acceleration structure build flags

- ドライバはアクセラレーション構造を、ツール（またはアプリ）がファイルに保存できる（まだ）不透明な形式にシリアライズします。

<table>
<thead>
<tr class="header">
<th><strong>#</strong></th>
<th><strong>PREFER_</strong>
<strong>FAST_</strong>
<strong>TRACE</strong></th>
<th><strong>PREFER_</strong>
<strong>FAST_</strong>
<strong>BUILD</strong></th>
<th><strong>ALLOW_</strong>
<strong>UPDATE</strong></th>
<th><strong>Properties</strong></th>
<th><strong>Example</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td><strong>1</strong></td>
<td>no</td>
<td>yes</td>
<td>no</td>
<td>Fastest possible build.<br />
Slower trace than #3 and #4.</td>
<td>Fully dynamic geometry like particles, destruction, changing prim counts or moving wildly (explosions etc), where per-frame rebuild is required.</td>
</tr>
<tr>
<td><strong>2</strong></td>
<td>no</td>
<td>yes</td>
<td>yes</td>
<td>Slightly slower build than #1, but allows very fast update.</td>
<td>Lower LOD dynamic objects, unlikely to be hit by too many rays but still need to be refitted per frame to be correct.</td>
</tr>
<tr>
<td><strong>3</strong></td>
<td>yes</td>
<td>no</td>
<td>no</td>
<td>Fastest possible trace.<br />
Slower build than #1 and #2.</td>
<td>Default choice for static level geometry.</td>
</tr>
<tr>
<td><strong>4</strong></td>
<td>yes</td>
<td>no</td>
<td>yes</td>
<td>Fastest trace against update-able acceleration structure. <br />
Updates slightly slower than #2.<br />
Trace a bit slower than #3.</td>
<td>Hero character, high-LOD dynamic objects that are expected to be hit by a significant number of rays.</td>
</tr>
</tbody>
</table>
</small>
- Then consider adding these flags:
  
  **ALLOW_COMPACTION**
  
  Whenever compaction is desired. It's generally a good idea to do
  this on all static geometry to reclaim (potentially significant)
  amounts of memory.
  
  For updateable geometry, it makes sense to compact those BVHs that
  have a long lifetime, so the extra step is worth it (compaction and
  update are not mutually exclusive\!).
  
  For fully dynamic geometry that's rebuilt every frame (as opposed to
  updated), there's generally no benefit from using compaction.
  
  One potential reason to NOT use compaction is to exploit the
  guarantee of BVH storage requirements increasing monotonically with
  primitive count -- this does not hold true in the context of
  compaction.
  
  **MINIMIZE_MEMORY**
  
  Use only when under general mem pressure, e.g. if otherwise a DXR
  path won't run at all because things don't fit. Usually costs build
  and trace perf.
---

# Determining raytracing support

CheckFeatureSupport() および D3D12_RAYTRACING_TIER を参照してください。これは、デバイスのレイトレーシングのサポートレベルを報告します。

[Raytracing emulation](#raytracing-emulation) is completely independent
of the above -- it is just a software library that sits on top of D3D.

---

## Raytracing emulation

> The following emulation feature proved useful during the experimental
> phase of DXR design. But as of the first shipping release of DXR, the
> plan is to stop maintaining this codebase. The cost/benefit is not
> justified and further, over time as more native DXR support comes
> online, the value of emulation will diminish further. That said, the
> description of what was done is left in case a strong justification
> crops up to resurrect the feature.

レイトレーシング フォールバック レイヤーは、DX12 コンピュートシェーダー ベースのソリューションを使用して、ネイティブ ドライバー/ハードウェアのサポートがないデバイスでのレイトレーシングのサポートを提供する ライブラリーです。このライブラリは、DX12 API のラッパーとして構築されており、DXR API とは異なる（しかし類似した）インターフェイスを備えています。また、このライブラリは、ドライバのサポートがある場合は DXR API を使用し、サポートがない場合はコンピュートへのフォールバックを可能にする内部スイッチを持っています。望ましい結果は、少なくとも従来のグラフィックスベースのレンダリング技術を補完／サポートする限定的なシナリオのために、レイトレーシングのネイティブサポートがない（そしてドライバが実装したエミュレーションがない）既存のハードウェアでフォールバックが有用であることです。また、エミュレーションは、GPU 上で動作するリファレンス実装として、レイトレーシングが可能なデバイスと比較することができます（WARP ソフトウェアラスターライザとは対照的）。

フォールバック層は、GitHub の公開レポを通じて提供され、開発者はこのライブラリをビルドして自身のエンジンに組み入れることができます。このレポは、エンジン開発者とハードウェア開発者の両方が、コードベースを改善するためのプルリクエストを提出できるように公開される予定です。フォールバック層を OS の外に置くことの利点は、DXIL コンパイラのリリースに合わせてスナップすることができ、開発者は自分のコードベースに特有の最適化でフォールバック層のスナップを自由にカスタマイズすることができることである。フォールバック層は、DXR API をサポートしていない古い Windows OS でも使用することができる。

この実装では、一連のコンピュートシェーダを使用して、GPU 時間軸上に加速構造を構築します。再帰的なシェーダの呼び出し（TraceRay または CallShader 経由）は、すべてのシェーダを大きなステートマシンシェーダにリンクして処理し、これらの呼び出しを関数呼び出しとしてエミュレートし、パラメータは GPU に割り当てられたスタックに保存される予定です。性能はまだ未定ですが、目標は、一般的なケースで小規模な技術に十分な性能が得られるようにすることです。

フォールバックレイヤーによるツールは、PIX で動作します。しかし、PIX のレイトレーシング固有のデバッグツール（例えば加速構造の可視化）の使用は、ドライバサポートのあるハードウェアに限定されます。フォールバックレイヤーのレイトレーシングの呼び出しは、基礎となるコンピュートシェーダのディスパッチとして表示されます。

フォールバックレイヤーインタフェースは、オリジナルの DXR API に忠実であろうとしていますが、いくつかの点で相違があります。主な違いは、DXR がシェーダでポインタを読み取ることを要求していることで、主にレイディスパッチでトップレベルのアクセラレーション構造からボトムレベルのアクセラレーション構造へトラバースする際に発生します。これを緩和するために、フォールバック層は開発者にポインタを GPU VA としてではなく、「記述子ヒープインデックス」と「バイトオフセット」のペアとして提供するよう強制し、エミュレートポインタと呼ばれる。これらは、ローカルルート記述子や array-of-pointer レイアウトの処理にも必要です。これにより、ポインタのサポートが抽象化され、ネイティブの DXR API からの移植が最小限に抑えられると期待されています。

---

# Tools support

デバッグレイヤーや PIX などのツールでキャプチャ、再生、解析ができるように、設計の一部が調整されました。以下に示す設計上の微調整に加えて、シェーダーパッチング、ルートシグネチャパッチング、より一般的には API フッキングなど、ツールによって適用可能ないくつかの汎用技術（レイトレーシングに特化したものではありません）もあります。

---

## Buffer bounds tracking

- [DispatchRays()](#dispatchrays) has input parameters are
  pointers (GPUVA) to shader tables. Size parameters are also present
  so that tools can tell how much memory is being used.

---

## Acceleration structure processing

- [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo) and
  [CopyRaytracingAccelerationStructure()](#copyraytracingaccelerationstructure) support dedicated modes for
  tools to operate on acceleration structures in the following
  ways:

  - **serialization:**

    ドライバは、キャプチャしたアプリケーションを後で再生する際に、上記のシリアライズされたフォーマットをデシリアライズします。その結果、シリアル化されたときにオリジナルと同じように機能し、シリアル化前のオリジナル構造と同じサイズかより小さい acceleration structure が得られます。これは、シリアライズが行われたのと同じデバイス／ドライバ上でのみ動作します。

  - **deserialization:**

    不透明なアクセラレーション構造を、ツールで可視化できる形に変換する。これは、acceleration structure の構築の逆のようなもので、この場合の出力は、不透明でないジオメトリやバウンディングボックスです。ツールは、アプリケーション実行中のどの時点でも、acceleration structure のビルド方法を追跡するオーバーヘッドを発生させることなく、acceleration structure の視覚化を表示することができます。

  - **visualization:**

    出力の形式は、以下のように、アプリケーションが acceleration structure を生成するために元々使用した入力と正確に一致しない場合があります。

    三角形の場合、視覚化のための出力は、acceleration structure 構築仕様で許可された次数依存性または他のバリエーションを除き、アプリケーションのオリジナルの acceleration structure と同じジオメトリのセットを表します。変換行列はジオメトリに折り込まれている可能性があります。三角形のフォーマットは（精度を落とすことなく）異なる可能性があります（アプリケーションが float16 データを使用していた場合、可視化の出力は float32 データになるかもしれません）。

    AABB の場合、元の AABB のセットに含まれる空間ボリュームは、出力 AABB のセットにも含まれなければなりませんが、AABBS の数が多いか少ないかで、より大きなボリュームをカバーすることができます。

    可視化には、OS が開発者モードであることが必要です。

    配置された acceleration structure のリソース状態の要件に関する議論については、acceleration structure 更新の制約を参照してください。これらの制約と、アクセラレーション構造体のすべての操作は、それを操作するための専用の API を経由する必要があるという事実が組み合わさって、PIX はアクセラレーション構造体の内容が有効であることを堅牢に信頼できることを意味します。

D3D12 がアプリケーションの実行にまたがる割り当てのための反復可能な VA 割り当てをサポートするようになったとしても、シリアライゼーションとデシリアライゼーションが PIX によって必要とされる可能性があることに注意してください。PIX が再生中にワークロードを変更したい場合、VA を維持することはできません。

- D3D ランタイムは、既存のステートオブジェクトの正しさを再確認する必要はありません。追加されるものが有効で、すでにステートオブジェクトにあるものと衝突しないことをチェックすればよいのです。

---

# API

---

## Device methods

D3D12 デバイスインターフェイスのセマンティクスにより、これらのデバイスメソッドは複数のスレッドから同時に呼び出すことができる。

---

### CheckFeatureSupport

```C++
HRESULT CheckFeatureSupport(
    D3D12_FEATURE Feature,
    [annotation("_Inout_updates_bytes_(FeatureSupportDataSize)")]
    void* pFeatureSupportData,
    UINT FeatureSupportDataSize
    );
```

これはレイトレーシングに特化した API ではなく、単に機能サポートを問い合わせるための一般的な D3D API です。レイトレーシングのサポートを問い合わせるには、Feature に `D3D12_FEATURE_D3D12_OPTIONS5` を渡し、pFeatureSupportData に D3D12_FEATURE_D3D12_OPTIONS5 変数を指定します。これはメンバ D3D12_RAYTRACING_TIER RaytracingTier を持っています。

---

#### CheckFeatureSupport Structures

---

##### D3D12_FEATURE_D3D12_OPTIONS5

```C++
// D3D12_FEATURE_D3D12_OPTIONS5
typedef struct D3D12_FEATURE_DATA_D3D12_OPTIONS5
{
    [annotation("_Out_")] BOOL SRVOnlyTiledResourceTier3;
    [annotation("_Out_")] D3D12_RENDER_PASS_TIER RenderPassesTier;
    [annotation("_Out_")] D3D12_RAYTRACING_TIER RaytracingTier;
} D3D12_FEATURE_DATA_D3D12_OPTIONS5;
```

レイトレーシングのサポートレベルである RaytracingTier を報告する D3D12 オプション構造体です（他の未リリース機能との間で）。D3D12_RAYTRACING_TIER を参照してください。

---

##### D3D12_RAYTRACING_TIER

```C++
typedef enum D3D12_RAYTRACING_TIER
{
    D3D12_RAYTRACING_TIER_NOT_SUPPORTED = 0,
    D3D12_RAYTRACING_TIER_1_0 = 10,
    D3D12_RAYTRACING_TIER_1_1 = 11,
} D3D12_RAYTRACING_TIER;
```

デバイス上のレイトレーシングのサポートレベル。CheckFeatureSupport()を介して問い合わせられます。

ValueDefinition`D3D12_RAYTRACING_TIER_NOT_SUPPORTED`No support for raytracing on the device. Attempts to create any raytracing related object will fail and using raytracing related APIs on command lists results in undefined behavior.`D3D12_RAYTRACING_TIER_1_0`The device supports the full raytracing functionality described in this spec, except features added in higher tiers listed below.`D3D12_RAYTRACING_TIER_1_1`Adds: <li>Support for indirect DispatchRays() calls (ray generation shader invocation) via [ExecuteIndirect()](#executeindirect).</li><li>Support for [incremental additions to existing state objects](#incremental-additions-to-existing-state-objects) via [AddToStateObject()](#addtostateobject).<li>Support for [inline raytracing](#inline-raytracing) via [RayQuery](#rayquery) objects declarable in any shader stage.</li><li>[GeometryIndex()](#geometryindex) intrinsic added to relevant raytracing shaders, for applications that wish to distinguish geometries manually in shaders in addition to or instead of by burning shader table slots.</li><li>Additional [ray flags](#ray-flags), `RAY_FLAG_SKIP_TRIANGLES` and `RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES`.</li><li>New version of raytracing pipeline config subobject, [D3D12_RAYTRACING_PIPELINE_CONFIG1](#d3d12_raytracing_pipeline_config1), adding a flags field, [D3D12_RAYTRACING_PIPELINE_FLAGS](#d3d12_raytracing_pipeline_flags). The equivalent subobject in HLSL is [RaytracingPipelineConfig1](#raytracing-pipeline-config1). The available flags, `D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_TRIANGLES` and `D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_PROCEDURAL_PRIMITIVES` (minus `D3D12_` when defined in HLSL) behave like OR'ing the equivalent RAY_FLAGS above into any [TraceRay()](#traceray) call in a raytracing pipeline, except that these do not show up in a [RayFlags()](#rayflags) call from a shader. Implementations may be able to make pipeline optimizations knowing that one of the primitive types can be skipped.</li><li>Additional vertex formats supported for acceleration structure build input as part of [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc).</li>---

### CreateStateObject

```C++
HRESULT CreateStateObject(
    _In_ const D3D12_STATE_OBJECT_DESC* pDesc,
    _In_ REFIID riid, // ID3D12StateObject
    _COM_Outptr_ void** ppStateObject
    );
```

概要については、State オブジェクトを参照してください。

ParameterDefinition`const D3D12_STATE_OBJECT_DESC* pDesc`Description of state object to create. See [D3D12_STATE_OBJECT_DESC](#d3d12_state_object_desc). To help generate this see the `CD3D12_STATE_OBJECT_DESC` helper in class in d3dx12.h.`REFIID riid`\__uuidof(ID3D12StateObject)`\_COM_Outptr_ void\*\* ppStateObject` Returned state object.```Return: HRESULT``S_OK```for success. `E_INVALIDARG`,`E_OUTOFMEMORY` on failure. The debug layer provides detailed status information.---

#### CreateStateObject Structures

ステートオブジェクトを定義するための以下の構造をより簡単に使用するためのヘルパー/サンプルラッパーコードが利用可能です。

---

##### D3D12_STATE_OBJECT_DESC

```C++
typedef struct D3D12_STATE_OBJECT_DESC
{
    D3D12_STATE_OBJECT_TYPE Type;
    UINT NumSubobjects;
    _In_reads_(NumSubobjects) const D3D12_STATE_SUBOBJECT* pSubobjects;
} D3D12_STATE_OBJECT_DESC;
```

MemberDefinition`D3D12_STATE_OBJECT_TYPE Type`See [D3D12_STATE_OBJECT_TYPE](#d3d12_state_object_type).`UINT NumSubobjects`Size of pSubobjects array.`_In_reads(NumSubobjects) const D3D12_STATE_SUBOBJECT* pSubobjects`Array of subobject definitions. See [D3D12_STATE_SUBOBJECT](#d3d12_state_subobject).---

##### D3D12_STATE_OBJECT_TYPE

```C++
typedef enum D3D12_STATE_OBJECT_TYPE
{
    D3D12_STATE_OBJECT_TYPE_COLLECTION = 0,
    // Could be added in future: D3D12_STATE_OBJECT_TYPE_COMPUTE_PIPELINE = 1,
    // Could be added in future: D3D12_STATE_OBJECT_TYPE_GRAPHICS_PIPELINE = 2,
    D3D12_STATE_OBJECT_TYPE_RAYTRACING_PIPELINE = 3,
} D3D12_STATE_OBJECT_TYPE;
```

ValueDefinition`D3D12_STATE_OBJECT_TYPE_COLLECTION`[Collection state object](#collection-state-object).`D3D12_STATE_OBJECT_TYPE_RAYTRACING_PIPELINE`[Raytracing pipeline state object](#raytracing-pipeline-state-object).---

##### D3D12_STATE_SUBOBJECT

```C++
typedef struct D3D12_STATE_SUBOBJECT
{
    D3D12_STATE_SUBOBJECT_TYPE Type;
    const void* pDesc;
} D3D12_STATE_SUBOBJECT;
```

ステートオブジェクトの記述の中にあるサブオブジェクト。

ParameterDefinition`D3D12_STATE_SUBOBJECT_TYPE Type`See [D3D12_STATE_SUBOBJECT_TYPE](#d3d12_state_subobject_type).`const void* pDesc`Pointer to state object description of the specified type.---

##### D3D12_STATE_SUBOBJECT_TYPE

```C++
typedef enum D3D12_STATE_SUBOBJECT_TYPE
{
  D3D12_STATE_SUBOBJECT_TYPE_STATE_OBJECT_CONFIG = 0,
  D3D12_STATE_SUBOBJECT_TYPE_GLOBAL_ROOT_SIGNATURE = 1,
  D3D12_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE = 2,
  D3D12_STATE_SUBOBJECT_TYPE_NODE_MASK = 3,
  // 4 unused
  D3D12_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY = 5,
  D3D12_STATE_SUBOBJECT_TYPE_EXISTING_COLLECTION = 6,
  D3D12_STATE_SUBOBJECT_TYPE_SUBOBJECT_TO_EXPORTS_ASSOCIATION = 7,
  D3D12_STATE_SUBOBJECT_TYPE_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION = 8,
  D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_SHADER_CONFIG = 9,
  D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_PIPELINE_CONFIG = 10,
  D3D12_STATE_SUBOBJECT_TYPE_HIT_GROUP = 11,
  D3D12_STATE_SUBOBJECT_TYPE_MAX_VALID,
} D3D12_STATE_SUBOBJECT_TYPE;
```

サブオブジェクトのタイプのセットで、それぞれに対応する構造体の定義があります。

ParameterDefinition`D3D12_STATE_SUBOBJECT_TYPE_STATE_OBJECT_CONFIG`[D3D12_STATE_OBJECT_CONFIG](#d3d12_state_object_config)`D3D12_STATE_SUBOBJECT_TYPE_GLOBAL_ROOT_SIGNATURE`[D3D12_GLOBAL_ROOT_SIGNATURE](#d3d12_global_root_signature)`D3D12_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE`[D3D12_LOCAL_ROOT_SIGNATURE](#d3d12_local_root_signature)`D3D12_STATE_SUBOBJECT_TYPE_NODE_MASK`[D3D12_NODE_MASK](#d3d12_node_mask)`D3D12_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY`[D3D12_DXIL_LIBRARY_DESC](#d3d12_dxil_library_desc)`D3D12_STATE_SUBOBJECT_TYPE_EXISTING_COLLECTION`[D3D12_EXISTING_COLLECTION_DESC](#d3d12_existing_collection_desc)`D3D12_STATE_SUBOBJECT_TYPE_SUBOBJECT_TO_EXPORTS_ASSOCIATION`[D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION](#d3d12_subobject_to_exports_association)`D3D12_STATE_SUBOBJECT_TYPE_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION`[D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION](#d3d12_dxil_subobject_to_exports_association)`D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_SHADER_CONFIG`[D3D12_RAYTRACING_SHADER_CONFIG](#d3d12_raytracing_shader_config)`D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_PIPELINE_CONFIG`[D3D12_RAYTRACING_PIPELINE_CONFIG](#d3d12_raytracing_pipeline_config)`D3D12_STATE_SUBOBJECT_TYPE_HIT_GROUP`[D3D12_HIT_GROUP_DESC](#d3d12_hit_group_desc)---

##### D3D12_STATE_OBJECT_CONFIG

```C++
typedef struct D3D12_STATE_OBJECT_CONFIG
{
    D3D12_STATE_OBJECT_FLAGS Flags;
} D3D12_STATE_OBJECT_CONFIG;
```

これは、シェーダーエクスポートと[関連付ける](#subobject-association-behavior)ことができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、[こちら](#subobject-association-requirements)を参照してください。

このサブオブジェクトは、状態オブジェクトの一般的なプロパティを定義する。状態オブジェクトにこのサブオブジェクトが存在するかどうかは**任意**です。存在する場合、状態オブジェクト内のすべてのエクスポートは、同じサブオブジェクト（または一致する定義を持つもの）に関連付けられなければなりません。この一貫性の要件は、以下に詳述する `D3D12_STATE_OBJECT_FLAG_ALLOW_STATE_OBJECT_ADDITIONS` フラグの存在を除き、より大きなステートオブジェクトに含まれる既存のコレクションには適用されません。

MemberDefinition`D3D12_STATE_OBJECT_FLAGS Flags`See [D3D12_STATE_OBJECT_FLAGS](#d3d12_state_object_flags).---

##### D3D12_STATE_OBJECT_FLAGS

```C++
typedef enum D3D12_STATE_OBJECT_FLAGS
{
    D3D12_STATE_OBJECT_FLAG_NONE = 0x0,
    D3D12_STATE_OBJECT_FLAG_ALLOW_LOCAL_DEPENDENCIES_ON_EXTERNAL_DEFINITONS = 0x1,
    D3D12_STATE_OBJECT_FLAG_ALLOW_EXTERNAL_DEPENDENCIES_ON_LOCAL_DEFINITIONS = 0x2,
    D3D12_STATE_OBJECT_FLAG_ALLOW_STATE_OBJECT_ADDITIONS = 0x4,
} D3D12_STATE_OBJECT_FLAGS;
```

ValueDefinition`D3D12_STATE_OBJECT_FLAG_ALLOW_LOCAL_DEPENDENCIES_ON_EXTERNAL_DEFINITONS`<p>This applies to state objects of type collection only, ignored otherwise.</p><p>The exports from this collection are allowed to have unresolved references (dependencies) that would have to be resolved (defined) when the collection is included in a containing state object (e.g. RTPSO). This includes depending on an externally defined subobject associations to associate an external subobject (e.g. root signature) to a local export.</p><p>In the absence of this flag (**default**), all exports in this collection must have their dependencies fully locally resolved, including any necessary subobject associations being defined locally. Advanced implementations/drivers will have enough information to compile the code in the collection and not need to keep around any uncompiled code (unless the `D3D12_STATE_OBJECT_FLAG_ALLOW_EXTERNAL_DEPENDENCIES_ON_LOCAL_DEFINITIONS` flag is set). So that when the collection is used in a containing state object (e.g. RTPSO), minimal work needs to be done by the driver, ideally a "cheap" link at most.</p><p>Even with this flag, there is never visibility of code across separate state object definitions that are combined incrementally via the [AddToStateObject()](#addtostateobject) API.</p>`D3D12_STATE_OBJECT_FLAG_ALLOW_EXTERNAL_DEPENDENCIES_ON_LOCAL_DEFINITIONS`<p>This applies to state objects of type collection only, ignored otherwise.</p><p>If a collection is included in another state object (e.g. RTPSO), allow shaders / functions in the rest of the containing state object to depend on (e.g. call) exports from this collection.</p><p> In the absence of this flag (default), exports from this collection cannot be directly referenced by other parts of containing state objects (e.g. RTPSO). This can reduce memory footprint for the collection slightly since drivers don't need to keep uncompiled code in the collection on the off chance that it may get called by some external function that would then compile all the code together. That said, if not all necessary subobject associations have been locally defined for code in this collection, the driver may not be able to compile shader code yet and may still need to keep uncompiled code around.</p><p>A subobject association defined externally that associates an external subobject to a local export does not count as an external dependency on a local definition, so the presence or absence of this flag does not affect whether the association is allowed or not. On the other hand if the current collection defines a subobject association for a locally defined subobject to an external export (e.g. shader), that counts as an external dependency on a local definition, so this flag must be set.</p><p>Also, regardless of the presence or absence of this flag, shader entrypoints (such as hit groups or miss shaders) in the collection are visible as entrypoints to a containing state object (e.g. RTPSO) if exported by it. In the case of an RTPSO, the exported entrypoints can be used in shader tables for raytracing.</p><p>Even with this flag, there is never visibility of code across separate state object definitions that are combined incrementally via the [AddToStateObject()](#addtostateobject) API.</p>`D3D12_STATE_OBJECT_FLAG_ALLOW_STATE_OBJECT_ADDITIONS`<p>The presence of this flag in an executable state object, e.g. raytracing pipeline, allows the state object to be passed into [AddToStateObject()](#addtostateobject) calls, either as the original state object, or the portion being added.</p><p>The presence of this flag in a collection state object means the collection can be imported by executable state objects (e.g. raytracing pipelines) regardless of whether they have also set this flag. The absence of this flag in a collection state object means the collection can only be imported by executable state objects that also do not set this flag.---

##### D3D12_GLOBAL_ROOT_SIGNATURE

```C++
typedef struct D3D12_GLOBAL_ROOT_SIGNATURE
{
    ID3D12RootSignature* pGlobalRootSignature;
} D3D12_GLOBAL_ROOT_SIGNATURE;
```

これは、シェーダーエクスポートと[関連付ける](#subobject-association-behavior)ことができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、[こちら](#subobject-association-requirements)を参照してください。

このサブオブジェクトは、関連付けられたシェーダで使用されるグローバル ルート シグネチャを定義します。ステート オブジェクトにこのサブオブジェクトが存在するかどうかは、オプションです。任意のシェーダー関数に関連付けられたグローバル/ローカル ルート署名の組み合わせは、シェーダーによって宣言されたすべてのリソース バインディングを定義する必要があります（グローバル ルート署名とローカル ルート署名の間で重複がないように）。

コールグラフ内の任意の関数が特定のグローバルルートシグネチャと関連付けられている場合、グラフ内の他の関数は同じグローバルルートシグネチャと関連付けられているか、何も関連付けられていないかのいずれかでなければならず、シェーダーエントリー（コールグラフのルート）はグローバルルートシグネチャと関連付けられていなければなりません。

しかし、CommandList から特定の DispatchRays() 操作の間に参照されるシェーダは、計算ルート署名として CommandList に設定されたものと同じグローバル・ルート署名を指定しなければなり ません。したがって、シェーダの異なるサブセットに関連付けられた複数のグローバルルートシグネチャを持つ単一の大きなステートオブジェクトを定義することは有効です。一部のシェーダが異なるグローバルルートシグネチャを使用するからといって、アプリがステートオブジェクトを分割することを強制されるわけではありません。

MemberDefinition`ID3D12RootSignature* pGlobalRootSignature`Root signature that will function as a global root signature. State object holds a reference.---

##### D3D12_LOCAL_ROOT_SIGNATURE

```C++
typedef struct D3D12_LOCAL_ROOT_SIGNATURE
{
    ID3D12RootSignature* pLocalRootSignature;
} D3D12_LOCAL_ROOT_SIGNATURE;
```

これは、シェーダーエクスポートと関連付けることができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、こちらを参照してください。

このサブオブジェクトは、関連付けられたシェーダーで使用されるローカルルートシグネチャを定義します。ステート オブジェクトにこのサブオブジェクトが存在するかどうかは、オプションです。任意のシェーダー関数に関連付けられたグローバル/ローカル ルート署名の組み合わせは、シェーダーによって宣言されたすべてのリソース バインディングを定義する必要があります（グローバルおよびローカル ルート署名の間で重複がないこと）。

コールグラフ内の任意の関数（シェーダーテーブル間のコールはカウントしない）が特定のローカルルート署名と関連付けられている場合、グラフ内の他の関数は同じローカルルート署名と関連付けられているか、何も関連付けられていないかのいずれかでなければならず、シェーダー項目（コールグラフのルート）はローカルルート署名と関連付けられていなければなりません。

これは、与えられたシェーダーエントリから到達可能なコードのセットは、シェーダーレコードのシェーダー識別子から呼び出されるという事実に対応しており、ローカルルート引数の単一のセットが適用されます。もちろん、異なるシェーダは異なるローカルルート引数を使うことができ ます（あるいは、使わないこともできます）。

MemberDefinition`ID3D12RootSignature* pLocalRootSignature`Root signature that will function as a local root signature. State object holds a reference.---

##### D3D12_DXIL_LIBRARY_DESC

```C++
typedef struct D3D12_DXIL_LIBRARY_DESC
{
    D3D12_SHADER_BYTECODE DXILLibrary;
    UINT NumExports;
    _In_reads_(NumExports) D3D12_EXPORT_DESC* pExports;
} D3D12_DXIL_LIBRARY_DESC;
```

MemberDefinition`D3D12_SHADER_BYTECODE DXILLibrary`Library to include in the state object. Must have been compiled with library target 6.3 or higher. It is fine to specify the same library multiple times either in the same state object / collection or across multiple, as long as the names exported each time don't conflict in a given state object.`UINT NumExports`Size of pExports array. If 0, everything gets exported from the library.`_In_reads(NumExports) D3D12_EXPORT_DESC* pExports`Optional exports array. See [D3D12_EXPORT_DESC](#d3d12_export_desc).---

##### D3D12_EXPORT_DESC

```C++
typedef struct D3D12_EXPORT_DESC
{
    LPCWSTR Name;
    _In_opt_ LPCWSTR ExportToRename;
    D3D12_EXPORT_FLAGS Flags;
} D3D12_EXPORT_DESC;
```

MemberDefinition`LPWSTR Name`<p>Name to be exported. If the name refers to a function that is overloaded, a mangled version of the name (function parameter information encoded in name string) can be provided to disambiguate which overload to use. The mangled name for a function can be retrieved from HLSL compiler reflection (not documented in this spec).</p><p> If ExportToRename field is non-null, Name refers to the new name to use for it when exported. In this case Name must be an unmangled name, whereas ExportToRename can be either a mangled or unmangled name. A given internal name may be exported multiple times with different renames (and/or not renamed).</p>`_In_opt_ LPWSTR ExportToRename`If non-null, this is the name of an export to use but then rename when exported. Described further above.`D3D12_EXPORT_FLAGS Flags`Flags to apply to the export.---

##### D3D12_EXPORT_FLAGS

```C++
typedef enum D3D12_EXPORT_FLAGS
{
    D3D12_EXPORT_FLAG_NONE = 0x0,
} D3D12_EXPORT_FLAGS;
```

現在、エクスポートフラグは定義されていません。

---

##### D3D12_EXISTING_COLLECTION_DESC

```C++
typedef struct D3D12_EXISTING_COLLECTION_DESC
{
    ID3D12StateObject* pExistingCollection;
    UINT NumExports;
    _In_reads_(NumExports) D3D12_EXPORT_DESC* pExports;
} D3D12_EXISTING_COLLECTION_DESC;
```

MemberDefinition`ID3D12StateObject* pExistingCollection`[Collection](#collection-state-object) to include in state object. Enclosing state object holds a ref on the existing collection.`UINT NumExports`Size of pExports array. If 0, all of the collection's exports get exported.`_In_reads(NumExports) D3D12_EXPORT_DESC* pExports`Optional exports array. See [D3D12_EXPORT_DESC](#d3d12_export_desc).---

##### D3D12_HIT_GROUP_DESC

```C++
typedef struct D3D12_HIT_GROUP_DESC
{
    LPWSTR HitGroupExport;
    D3D12_HIT_GROUP_TYPE Type;
    _In_opt_ LPWSTR AnyHitShaderImport;
    _In_opt_ LPWSTR ClosestHitShaderImport;
    _In_opt_ LPWSTR IntersectionShaderImport;
} D3D12_HIT_GROUP_DESC;
```

MemberDefinition`LPWSTR HitGroupExport`Name to give to hit group.`D3D12_HIT_GROUP_TYPE Type`See [D3D12_HIT_GROUP_TYPE](#d3d12_hit_group_type).`_In_opt_ LPWSTR AnyHitShaderImport`Optional name of anyhit shader. Can be used with all hit group types.`_In_opt_ LPWSTR ClosestHitShaderImport`Optional name of closesthit shader. Can be used with all hit group types.`_In_opt_ LPWSTR IntersectionShaderImport`Optional name of intersection shader. Can only be used with hit groups of type procedural primitive.---

##### D3D12_HIT_GROUP_TYPE

```C++
typedef enum D3D12_HIT_GROUP_TYPE
{
    D3D12_HIT_GROUP_TYPE_TRIANGLES = 0x0,
    D3D12_HIT_GROUP_TYPE_PROCEDURAL_PRIMITIVE = 0x1,
} D3D12_HIT_GROUP_TYPE;
```

ヒットグループの種類を指定します。三角形の場合、ヒットグループには交差シェーダを含めることができません。プロシージャル・プリミティブの場合、ヒット・グループには交差点シェーダが含まれていなければなりません。

> This enum exists to allow the possibility in the future of having other
> hit group types which may not otherwise be distinguishable from the
> other members of `D3D12_HIT_GROUP_DESC`. For instance, a new type might
> simply change the meaning of the intersection shader import to represent
> a different formulation of procedural primitive.

---

##### D3D12_RAYTRACING_SHADER_CONFIG

```C++
typedef struct D3D12_RAYTRACING_SHADER_CONFIG
{
  UINT MaxPayloadSizeInBytes;
  UINT MaxAttributeSizeInBytes;
} D3D12_RAYTRACING_SHADER_CONFIG;
```

これは、シェーダーエクスポートと関連付けることができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、こちらを参照してください。

レイトレーシング パイプラインには、1 つのレイトレーシング シェーダ コンフィギュレーションが必要です。複数のシェーダ構成が存在する場合（たとえば、各コレクションに 1 つずつ入っていて、それぞれ独立したドライバのコンパイルを可能にする）、レイトレーシング パイプラインに結合したときにすべて一致する必要があります。

MemberDefinition`UINT MaxPayloadSizeInBytes`The maximum storage for scalars (counted as 4 bytes each) in ray payloads in raytracing pipelines that contain this program. Callable shader payloads are not part of this limit. This field is ignored for payloads that use [payload access qualifiers](#payload-access-qualifiers).`UINT MaxAttributeSizeInBytes`The maximum number of scalars (counted as 4 bytes each) that can be used for attributes in pipelines that contain this shader. The value cannot exceed [D3D12_RAYTRACING_MAX_ATTRIBUTE_SIZE_IN_BYTES](#constants).---

##### D3D12_RAYTRACING_PIPELINE_CONFIG

```C++
typedef struct D3D12_RAYTRACING_PIPELINE_CONFIG
{
    UINT MaxTraceRecursionDepth;
} D3D12_RAYTRACING_PIPELINE_CONFIG;
```

これは、シェーダーエクスポートと関連付けることができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、こちらを参照してください。

レイトレーシング パイプラインには、1 つのレイトレーシング パイプライン構成が必要です。複数のシェーダ構成が存在する場合（たとえば、それぞれのコレクションに 1 つずつ入っていて、それぞれ独立したドライバのコンパイルを可能にする）、レイトレーシング パイプラインに結合したときにそれらがすべて一致する必要があります。

MemberDefinition`UINT MaxTraceRecursionDepth`Limit on ray recursion for the raytracing pipeline. See [Ray recursion limit](#ray-recursion-limit).---

##### D3D12_RAYTRACING_PIPELINE_CONFIG1

```C++
typedef struct D3D12_RAYTRACING_PIPELINE_CONFIG1
{
    UINT MaxTraceRecursionDepth;
    D3D12_RAYTRACING_PIPELINE_FLAGS Flags;
} D3D12_RAYTRACING_PIPELINE_CONFIG1;
```

`D3D12_RAYTRACING_PIPELINE_CONFIG1` は Tier 1.1 のレイトレーシングをサポートする必要があります。

これは、シェーダーエクスポートと関連付けることができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、こちらを参照してください。

レイトレーシング パイプラインには、1 つのレイトレーシング パイプライン構成が必要です。複数のシェーダ構成が存在する場合（たとえば、それぞれのコレクションに 1 つずつ入っていて、それぞれ独立したドライバのコンパイルを可能にする）、レイトレーシング パイプラインに結合したときにそれらがすべて一致する必要があります。

MemberDefinition`UINT MaxTraceRecursionDepth`Limit on ray recursion for the raytracing pipeline. See [Ray recursion limit](#ray-recursion-limit).`D3D12_RAYTRACING_PIPELINE_FLAGS Flags`See [D3D12_RAYTRACING_PIPELINE_FLAGS](#d3d12_raytracing_pipeline_flags).---

##### D3D12_RAYTRACING_PIPELINE_FLAGS

```C++
typedef enum D3D12_RAYTRACING_PIPELINE_FLAGS
{
    D3D12_RAYTRACING_PIPELINE_FLAG_NONE                         = 0x0,
    D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_TRIANGLES               = 0x100,
    D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_PROCEDURAL_PRIMITIVES   = 0x200,
} D3D12_RAYTRACING_PIPELINE_FLAGS;

```

D3D12_RAYTRACING_PIPELINE_CONFIG1 の Flags メンバです。

ValueDefinition`D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_TRIANGLES`<p>For any [TraceRay()](#traceray) call within this raytracing pipeline, add in the `RAY_FLAG_SKIP_TRIANGLES` [Ray flag](#ray-flags). The resulting combination of ray flags must be valid. The presence of this flag in a raytracing pipeline config does not show up in a [RayFlags()](#rayflags) call from a shader. Implementations may be able to optimize pipelines knowing that a particular primitive type need not be considered.</p>`D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_PROCEDURAL_PRIMITIVES`<p>For any [TraceRay()](#traceray) call within this raytracing pipeline, add in the `RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES` [Ray flag](#ray-flags). The resulting combination of ray flags must be valid. The presence of this flag in a raytracing pipeline config does not show up in a [RayFlags()](#rayflags) call from a shader. Implementations may be able to optimize pipelines knowing that a particular primitive type need not be considered.</p>---

##### D3D12_NODE_MASK

```C++
typedef struct D3D12_NODE_MASK
{
    UINT NodeMask;
} D3D12_NODE_MASK;
```

これは、シェーダーエクスポートと関連付けることができるサブオブジェクトのタイプです。ステート オブジェクト内のさまざまなシェーダーに関連付けることができる/できなければならないサブオブジェクト タイプに関するルールの概要は、こちらを参照してください。

ノードマスクサブオブジェクトは、ステートオブジェクトがどの GPU ノードに適用されるかを識別します。これはオプションで、これがない場合、ステートオブジェクトは利用可能なすべてのノードに適用されます。ノードマスクサブオブジェクトがステートオブジェクトのいずれかの部分と関連付けられている場合、ノードマスクの関連付けはステートオブジェクト内のすべてのエクスポート（インポートされたコレクションを含む）に対して行われ、参照されるすべてのノードマスクサブオブジェクトが一致するコンテンツを持つ必要があります。

---

##### D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION

```C++
typedef struct D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION
{
  const D3D12_STATE_SUBOBJECT* pSubobjectToAssociate;
  UINT NumExports;
  _In_reads_(NumExports) LPCWSTR* pExports;
} D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION;
```

ステート オブジェクトで直接定義されたサブオブジェクトをシェーダー エクスポートと関連付けます。クロスリンケージを選択するためのオプションの D3D12_STATE_OBJECT_CONFIG サブオブジェクトのフラグの選択に応じて、関連付けられるエクスポートは必ずしも現在のステートオブジェクト（またはまだ見られているもの）に存在する必要はありません -- RTPSO 作成時など、後で解決されるようにします。詳しくは、サブオブジェクトの関連付けの動作を参照してください。

MemberDefinition`const D3D12_STATE_SUBOBJECT* pSubobjectToAssociate`Pointer to subobject in current state object to define an association to.`UINT NumExports`<p>Size of export array. If 0, this is being explicitly defined as a default association. See [Subobject association behavior](#subobject-association-behavior).</p><p>Another way to define a default association is to omit this subobject association for that subobject completely.</p>`_In_reads_(NumExports) LPCWSTR* pExports`Exports to associate subobject with.---

##### D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION

```C++
typedef struct D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION
{
    LPWCSTR pDXILSubobjectName;
    UINT NumExports;
    _In_reads_(NumExports) LPCWSTR* pExports;
} D3D12_DXIL_SUBOBJECT_TO_EXPORTS_ASSOCIATION;
```

DXIL ライブラリで定義されたサブオブジェクト（まだ見たことがないものでもよい）をシェーダーエクスポートと関連付けます。詳細は、「サブオブジェクトの関連付けの動作」を参照してください。

MemberDefinition`LPWSTR pDXILSubobjectName`Name of subobject defined in a DXIL library.`UINT NumExports`<p>Size of export array. If 0, this is being explicitly defined as a default association. See [Subobject association behavior](#subobject-association-behavior).</p><p>Another way to define a default association is to omit this subobject association for that subobject completely.</p>`_In_reads_(NumExports) LPCWSTR* pExports`Exports to associate subobject with.---

### AddToStateObject

```C++
HRESULT AddToStateObject(
    _In_ const D3D12_STATE_OBJECT_DESC* pAddition,
    _In_ ID3D12StateObject* pStateObjectToGrowFrom,
    _In_ REFIID riid, // ID3D12StateObject
    _COM_Outptr_ void** ppNewStateObject
    );
```

既存のステートオブジェクトにインクリメンタルに追加します。 これは、既存のオブジェクトのスーパーセットであるステートオブジェクトをゼロから作成する（例：シェーダーを少し追加する）よりも低い CPU オーバーヘッドを発生させます。 オーバーヘッドが低い理由は以下の通りです。

- ドライバは理想的には追加されたシェーダをコンパイルするだけでよく、追加されたものがたまたま既存のコレクションであった場合は、それさえも避けることができます。クリーンなドライバ実装では、ステートオブジェクト全体のための軽量の高レベルリンク手順が必要なだけです。

- 状態オブジェクトの記述は完全に自己完結していなければならない。例えば、既存の状態オブジェクトの内容を一切参照してはならない。別の言い方をすれば、その記述は、それ自体で CreateStateObject() に送信されたものとして有効でなければならない。

> It wasn't deemed worth the effort or complexity to support incremental deletion, i.e. DeleteFromStateObject\(\). If an app finds that state object memory footprint is such a problem that it needs to periodically trim by shrinking state objects, it has to create the desired smaller state objects from scratch. In this case, if existing [collections](#collection-state-object) are used to piece together the smaller state object, at least driver overhead will be reduced, if not runtime overhead of parsing/validating the new state object as a whole.

`AddToStateObject` は新しいステートオブジェクトを返すので、元のオブジェクトが不要になったとき（そして飛行中の作業で参照されなくなったとき）、アプリケーションは元のオブジェクトを解放することができる。 副産物として、 `AddToStateObject` を使用してステートオブジェクトの分岐した系統を作成することが有効であり、一般的には 1 つの系統しか継承されない可能性があります。 あるステートオブジェクトは、それが追加したエクスポートとその親が公開したものだけを公開し、兄弟が行うものは公開しません。 親からのシェーダ識別子は、その子に対して有効であり、再取得する必要はありません。

[State object lifetimes as seen by driver](#state-object-lifetimes-as-seen-by-driver) is a discussion useful for driver authors.

ランタイムは、ファミリー内のすべてのステートオブジェクトで共有ミューテックスを使用し、スレッドセーフを実施します。 `AddToStateObject` API はファミリー内のステートオブジェクトをライタロックします。 また、シェーダのエクスポートに基づく情報を取得する API、GetShaderIdentifier() および GetShaderStackSize() は、ファミリー内のステートオブジェクトに対して共有リーダーロックを取ります。

> A straightforward way to minimize any thread contention is to retrieve any needed shader identifiers etc. from a state object right after it is created (and remembering that any other state objects created from this one share identifiers so they need not be re-retrieved). So that by the time `AddToStateObject` needs to be called, nothing will be at risk of blocking (unless `AddToStateObject` itself is called in parallel with related state objects for some reason).

新しいステートオブジェクトは、以前と同じパイプラインスタックサイズ設定で開始されます。(ランタイムは、既存のステートオブジェクトが `AddToStateObject()` 呼び出しで使用されている間に、アプリが SetPipelineStackSize() を呼び出してスタックサイズを変更しても、スレッドセーフを実施しようとしません)。 `AddToStateObject()` が新しいステートオブジェクトを返した後、アプリは新しく追加されたシェーダで GetShaderStackSize() を呼び出し、新しいステートオブジェクトのパイプラインスタックサイズを SetPipelineStackSize() で更新する必要があるかどうかを決定することができます。

ステートオブジェクトへの追加は、ステートオブジェクトにすでに存在するもの（ランタイムによって検証される）にどのように関係しなければならないかについて、いくつかの規定があります。

- D3D12_RAYTRACING_PIPELINE_CONFIG などのグローバルサブオブジェクトの定義は、既存のステートオブジェクトで定義されている方法と一致していなければなりません（たとえば、同じ MaxTraceRecursionDepth）。

- 元の状態オブジェクトと追加される部分の両方が、AddToStateObject() で使用されることにオプトインしていなければなりません。これは、D3D12_STATE_OBJECT_CONFIG の flags メンバーの一部として `D3D12_STATE_OBJECT_FLAG_ALLOW_STATE_OBJECT_ADDITIONS` を指定することで実現できます。 `D3D12_STATE_OBJECT_FLAG_ALLOW_STATE_OBJECT_ADDITIONS` はコレクションステートオブジェクトに何を意味するかといった詳細については Flags を参照 してください。

- エクスポートされたシェーダーエントリポイント（すなわち、シェーダー識別子をサポートする）は、既存のステートオブジェクトからのエクスポートと衝突しないユニークな名前を持っている必要があります。

- ライブラリ関数などの非シェーダエントリポイントは、ステートオブジェクトにすでにあるものに関連して、（同じまたは異なるコード定義で名前を再利用して）繰り返し定義することが許可されています（ローカルで表示する必要がある場合に必要です）。同じことが、定義された様々なサブオブジェクトにも当てはまります。

- `D3D12_SRV_DIMENSION_RAYTRACING_ACCELERATION_STRUCTURE` (その記述は単に GPUVA です。下記参照)の次元の記述子ヒープベースの SRV を介して。

> Disallowing cross-visibility between the existing state object and what is being added offers several benefits. It simplifies the incremental compilation burden on the driver, e.g. it isn't forced to touch existing compiled code. It avoids messy semantic issues like the presence of a new subobject affecting [default associations](#default-associations) that may have applied to the existing state object (if cross visibility were possible). Finally it simplifies runtime validation code complexity.

ParameterDefinition`const D3D12_STATE_OBJECT_DESC* pAddition`Description of state object contents to add to existing state object. See [D3D12_STATE_OBJECT_DESC](#d3d12_state_object_desc). To help generate this see the `CD3D12_STATE_OBJECT_DESC` helper in class in d3dx12.h.`ID3D12StateObject* pStateObjectToGrowFrom`<p>Existing state object, which can be in use (e.g. active raytracing) during this operation.</p><p>The existing state object must **not** be of type [Collection](#collection-state-object) - it is deemed too complex to bother defining behavioral semantics for this.</p>`REFIID riid`\__uuidof(ID3D12StateObject)`\_COM_Outptr_ void** ppNewStateObject`<p>Returned state object.</p><p>Behavior is undefined if shader identifiers are retrieved for new shaders from this call and they are accessed via shader tables by any already existing or in flight command list that references some older state object. Use of the new shaders added to the state object can only occur from commands (such as DispatchRays or ExecuteIndirect calls) recorded in a command list **after\*\* the call to `AddToStateObject`.</p>` Return: HRESULT``S_OK ` for success. `E_INVALIDARG`, `E_OUTOFMEMORY` on failure. The debug layer provides detailed status information.---

### GetRaytracingAccelerationStructurePrebuildInfo

```C++
void GetRaytracingAccelerationStructurePrebuildInfo(
    _In_ const D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS* pDesc,
    _Out_ D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO *pInfo);
```

アクセラレーション構造体を構築するためのリソース要件をドライバに問い合わせる。入力される acceleration structure の記述は BuildRaytracingAccelerationStructure() に入るものと同じである。この関数の結果、アプリケーションは BuildRaytracingAccelerationStructure()に同じジオメトリで正しい量の出力ストレージとスクラッチストレージを提供することができます。

GetAccelerationStructurePrebuildInfo() に渡された同じ構成で、ジオメトリ数/インスタンス数および任意のジオメトリの頂点/インデックス/AABB 数が同じか小さい以外は、ビルドを実行することも可能です。この場合、GetRaytracingAccelerationStructurePrebuildInfo()に渡された元のサイズで報告されたストレージ要件が有効になります。これは、アクセラレーション構造に対して比較的大きなストレージを割り当てても問題ないようなアプリのシナリオに便利です。

このメソッドはコマンドリストではなくデバイス上にあり、ドライバは実際の頂点データ、インデックスデータなどを含む GPU メモリへのポインタを参照することなく、呼び出しの CPU 可視部分だけを見て、加速構造構築のリソース要件を計算できなければならないという前提に立ちます。

ParameterDefinition`const D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS* pDesc`<p>Description of the acceleration structure build. See [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS](#d3d12_build_raytracing_acceleration_structure_inputs). This structure is shared with [BuildRaytracingAccelerationStructure()](#buildraytracingaccelerationstructure).</p><p> The implementation is allowed to look at all the CPU parameters in this struct and nested structs. It may not inspect/dereference any GPU virtual addresses, other than to check to see if a pointer is NULL or not, such as the optional Transform in [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc), without dereferencing it.</p><p>In other words, the calculation of resource requirements for the acceleration structure does not depend on the actual geometry data (such as vertex positions), rather it can only depend on overall properties, such as the number of triangles, number of instances etc.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO* pInfo`Result ( [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO](#d3d12_raytracing_acceleration_structure_prebuild_info)) of query.---

#### GetRaytracingAccelerationStructurePrebuildInfo Structures

以下に加え、他の構造体（両 API に共通）については BuildRaytracingAccelerationStructure() を参照してください。

---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO
{
    UINT64 ResultDataMaxSizeInBytes;
    UINT64 ScratchDataSizeInBytes;
    UINT64 UpdateScratchDataSizeInBytes;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO;
```

MemberDefinition`UINT64 ResultDataMaxSizeInBytes`Size required to hold the result of an acceleration structure build based on the specified inputs.`UINT64 ScratchDataSizeInBytes`Scratch storage on GPU required during acceleration structure build based on the specified inputs.`UINT64 UpdateScratchDataSizeInBytes`<p>Scratch storage on GPU required during an acceleration structure update based on the specified inputs. This only needs to be called for the original acceleration structure build, and defines the scratch storage requirement for every acceleration structure update (other than the initial build).</p><p>If the `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_UPDATE` flag is not specified, this parameter returns 0.</p>---

### CheckDriverMatchingIdentifier

```C++
D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS
CheckDriverMatchingIdentifier(
  _In_ D3D12_SERIALIZED_DATA_TYPE SerializedDataType,
  _In_ const D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER* pIdentifierToCheck);
```

あるアプリがドライバによってシリアライズされたデータを持っているとします。特に、そのデータは、モード `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_SERIALIZE` で CopyRaytracingAccelerationStructure() への呼び出しから生じるシリアライズしたレイトレーシング加速構造で、おそらくアプリケーションの以前の実行から生じたものです。CheckDriverMatchingIdentifier() は、現在のデバイス/ドライバとシリアライズされたデータの互換性を報告します。

ParameterDefinition`D3D12_SERIALIZED_DATA_TYPE SerializedDataType`See [D3D12_SERIALIZED_DATA_TYPE](#d3d12_serialized_data_type).`const D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER* pIdentifierToCheck`Identifier from the header of the serialized data to check with the driver. See [D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER](#d3d12_serialized_data_driver_matching_identifier).`Return: D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS`See [D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS](#d3d12_driver_matching_identifier_status)---

#### CheckDriverMatchingIdentifier Structures

---

##### D3D12_SERIALIZED_DATA_TYPE

```C++
typedef enum D3D12_SERIALIZED_DATA_TYPE
{
    D3D12_SERIALIZED_DATA_RAYTRACING_ACCELERATION_STRUCTURE = 0x0,
} D3D12_SERIALIZED_DATA_TYPE;
```

シリアライズされたデータの種類。現時点では 1 つだけです。

ValueDefinition`D3D12_SERIALIZED_DATA_RAYTRACING_ACCELERATION_STRUCTURE`Serialized data contains a raytracing acceleration structure.---

##### D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER

```C++
typedef struct D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER
{
    GUID DriverOpaqueGUID;
    BYTE DriverOpaqueVersioningData[16];
} D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER;
```

シリアライズドアクセラレーション構造体のドライババージョンを記述する不透明なデータ構造です。これは、D3D12_SERIALIZED_ACCELERATION_STRUCTURE_HEADER というシリアライズされたアクセラレーション構造体のヘッダーのメンバである。この識別子を CheckDriverMatchingIdentifier() に渡すと、以前にシリアル化された acceleration structure が現在のドライバー/デバイスと互換性があり、したがって、デシリアライズしてレイトレーシングに使用できるかどうかをアプリに知らせます。

---

##### D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS

```C++
typedef enum D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS
{
    D3D12_DRIVER_MATCHING_IDENTIFIER_COMPATIBLE_WITH_DEVICE = 0x0,
    D3D12_DRIVER_MATCHING_IDENTIFIER_UNSUPPORTED_TYPE = 0x1,
    D3D12_DRIVER_MATCHING_IDENTIFIER_UNRECOGNIZED = 0x2,
    D3D12_DRIVER_MATCHING_IDENTIFIER_INCOMPATIBLE_VERSION = 0x3,
    D3D12_DRIVER_MATCHING_IDENTIFIER_INCOMPATIBLE_TYPE = 0x4,
} D3D12_DRIVER_MATCHING_IDENTIFIER_STATUS;
```

CheckDriverMatchingIdentifier()の戻り値です。

ValueDefinition`D3D12_DRIVER_MATCHING_IDENTIFIER_COMPATIBLE_WITH_DEVICE`Serialized data is compatible with the current device/driver.`D3D12_DRIVER_MATCHING_IDENTIFIER_UNSUPPORTED_TYPE`[D3D12_SERIALIZED_DATA_TYPE](#d3d12_serialized_data_type) specified is unknown or unsupported.`D3D12_DRIVER_MATCHING_IDENTIFIER_UNRECOGNIZED`Format of the data in [D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER](#d3d12_serialized_data_driver_matching_identifier) is unrecognized. This could indicate either corrupt data or the identifier was produced by a different hardware vendor.`D3D12_DRIVER_MATCHING_IDENTIFIER_INCOMPATIBLE_VERSION`Serialized data is recognized (likely from the same hardware vendor), but its version is not compatible with the current driver.`D3D12_DRIVER_MATCHING_IDENTIFIER_INCOMPATIBLE_TYPE`[D3D12_SERIALIZED_DATA_TYPE](#d3d12_serialized_data_type) specifies a data type that is not compatible with the type of serialized data. As long as there is only a single defined serialized data type this error cannot not be produced.---

### CreateCommandSignature

```C++
HRESULT ID3D12Device::CreateCommandSignature(
    const D3D12_COMMAND_SIGNATURE_DESC* pDesc,
    ID3D12RootSignature* pRootSignature,
    REFIID riid, // Expected: ID3D12CommandSignature
    void** ppCommandSignature
);
```

これはレイトレーシングに特化した API ではなく、ExecuteIndirect()で使用するコマンドシグネチャを作成するための一般的な D3D API に過ぎません。 このセクションで示されているのは、間接的な引数バッファから間接的な DispatchRays() 呼び出しを可能にするために追加されたものです。

特に、D3D12_COMMAND_SIGNATURE_DESC フィールドは、DispatchRays を有効にするために設定することができます。

---

#### CreateCommandSignature Structures

---

##### D3D12_COMMAND_SIGNATURE_DESC

```C++
typedef struct D3D12_COMMAND_SIGNATURE_DESC
{
    // The number of bytes between each drawing structure
    UINT ByteStride;
    UINT NumArgumentDescs;
    const D3D12_INDIRECT_ARGUMENT_DESC *pArgumentDescs;
    UINT NodeMask;
} D3D12_COMMAND_SIGNATURE_DESC;
```

CreateCommandSignature()を介してコマンドシグネチャを定義するための構造体です。 レイトレーシングとの関連は、D3D12_INDIRECT_ARGUMENT_DESC 配列は、その最後のエントリとして DispatchRays パラメータ用のエントリを持つことができます（そして、コマンドシグネチャの Draw または Dispatch 引数の使用とは相互排他的です）。

---

##### D3D12_INDIRECT_ARGUMENT_DESC

```C++
typedef struct D3D12_INDIRECT_ARGUMENT_DESC
{
    D3D12_INDIRECT_ARGUMENT_TYPE Type;
    ...
} D3D12_INDIRECT_ARGUMENT_DESC;
```

D3D12_COMMAND_SIGNATURE_DESC で間接引数を定義するための構造体です。 レイトレーシングの関連フィールドである D3D12_INDIRECT_ARGUMENT_TYPE のみを表示し、DispatchRays のエントリーがあります。

---

##### D3D12_INDIRECT_ARGUMENT_TYPE

```C++
typedef enum D3D12_INDIRECT_ARGUMENT_TYPE
{
    ...
    D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH_RAYS,
    ...
} D3D12_INDIRECT_ARGUMENT_TYPE;
```

D3D12_INDIRECT_ARGUMENT_DESC のパラメータ・タイプです。ここでは、レイトレーシング DispatchRays に関連する値のみを表示しています。

ValueDefinition`D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH_RAYS`<p>DispatchRays argument in command signature. Each command in the indirect argument buffer includes [D3D12_DISPATCH_RAYS_DESC](#d3d12_dispatch_rays_desc) values for the current indirect argument.</p><p>When this argument is used, the command signature can't change vertex buffer or index buffer bindings (as those are related to the Draw pipeline only).</p>---

## Command list methods

すべてのコマンドリストメソッドについて、コマンドリストの記録時にランタ イムはパラメータのディープコピーを作成します（GPU 仮想アドレスで指さ れる GPU メモリ内のデータは含まれません）。そのため、呼び出しが返されるときに、パラメータ用のアプリケーショ ンの CPU メモリはもう参照されません。コマンドが GPU タイムラインで実際に実行されるとき、GPU 仮想アド レスによって識別されるすべての GPU メモリがアクセスされ、コマンド リスト記録時間から独立して、アプリケーションがその メモリを変更する自由が与えられます。

---

### BuildRaytracingAccelerationStructure

```C++
void BuildRaytracingAccelerationStructure(
    _In_ const D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC* pDesc,
    _In_ UINT NumPostbuildInfoDescs,
    _In_reads_opt_(NumPostbuildInfoDescs)
        const D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC* pPostbuildInfoDescs
);
```

GPU 上で acceleration structure ビルドを実行します。また、オプションで、ビルドの直後にビルド後の情報を出力します。

> This postbuild information can also be obtained separately from an
> already built acceleration structure via [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo).
> The advantage of generating postbuild info along with a build is that a
> barrier isn't needed in between the build completing and requesting
> postbuild information, for the case where an app knows it needs the
> postbuild info right away.

概要については、ジオメトリとアクセラレーション構造を参照してください。

ルールと決定論の議論については、アクセラレーション構造のプロパティを参照してください。

また、アクセラレーション構造体を構築するための一般的なヒントも参照してください。

graphics や compute コマンドリストで呼び出せますが、bundle からは呼び出せません。

ParameterDefinition`const D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC* pDesc`Description of the acceleration structure to build. See [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC](#d3d12_build_raytracing_acceleration_structure_desc).`UINT NumPostbuildInfoDescs`Size of postbuild info desc array. Set to 0 if none are needed.`const D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC* pPostbuildInfoDescs`Optional array of descriptions for postbuild info to generate describing properties of the acceleration structure that was built. See [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_desc). Any given postbuild info type,[D3D12_RAYTRACING_ACCEELRATION_STRUCTURE_POSTBUILD_INFO_TYPE](#d3d12_raytracing_acceleration_structure_postbuild_info_type), can only be selected for output by at most one array entry.---

#### BuildRaytracingAccelerationStructure Structures

---

##### D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC

```C++
typedef struct D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC
{
    D3D12_GPU_VIRTUAL_ADDRESS DestAccelerationStructureData;
    D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS Inputs;
    _In_opt_ D3D12_GPU_VIRTUAL_ADDRESS SourceAccelerationStructureData;
    D3D12_GPU_VIRTUAL_ADDRESS ScratchAccelerationStructureData;
} D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC;
```

MemberDefinition`D3D12_GPU_VIRTUAL_ADDRESS DestAccelerationStructureData`<p> Location to store resulting acceleration structure. [GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo) reports the amount of memory required for the result here given a set of acceleration structure build parameters.</p><p> The address must be aligned to 256 bytes ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)).</p><p> The memory pointed to must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>`D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS Inputs`Description of the input data for the acceleration structure build. This is packaged in its own struct since it is shared with [GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo).`D3D12_GPU_VIRTUAL_ADDRESS SourceAccelerationStructureData`<p>Address of an existing acceleration structure if an acceleration structure update (incremental build) is being requested, by setting D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PERFORM_UPDATE in the Flags parameter. Otherwise this address must be NULL.</p><p> If this address is the same as DestAccelerationStructureData, the update is to be performed in-place. Any other form of overlap of the source and destination memory is invalid and produces undefined behavior.</p><p>The address must be aligned to 256 bytes ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)), which is a somewhat redundant requirement as any existing acceleration structure passed in here would have already been required to be placed with such alignment anyway.</p><p>The memory pointed to must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>`D3D12_GPU_VIRTUAL_ADDRESS ScratchAccelerationStructureData`<p>Location where the build will store temporary data.[GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo) reports the amount of scratch memory the implementation will need for a given set of acceleration structure build parameters.</p><p> The address must be aligned to 256 bytes ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)).</p><p> Contents of this memory going into a build on the GPU timeline are irrelevant and will not be preserved. After the build is complete on the GPU timeline, the memory is left with whatever undefined contents the build finished with.</p><p> The memory pointed to must be in state `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`.</p>---

##### D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS

```C++
typedef struct D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS
{
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE Type;
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS Flags;
    UINT NumDescs;
    D3D12_ELEMENTS_LAYOUT DescsLayout;
    union
    {
        D3D12_GPU_VIRTUAL_ADDRESS InstanceDescs;
        const D3D12_RAYTRACING_GEOMETRY_DESC* pGeometryDescs;
        const D3D12_RAYTRACING_GEOMETRY_DESC*const* ppGeometryDescs;
    };
} D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS;
```

この構造体は BuildRaytracingAccelerationStructure() と GetRaytracingAccelerationStructurePrebuildInfo() の両方によって使用されます。

実際にビルドを行わない GetRaytracingAccelerationStructurePrebuildInfo() では、 `D3D12_GPU_VIRTUAL_ADDRESS` （GPU メモリ内）経由で参照される任意のパラメータ、例えば InstanceDescs は操作によりアクセスされません。したがって、このメモリはまだ初期化する必要がなく、特定のリソースの状態である必要もありません。GPU アドレスがヌルであるかどうかは、ポインタが再参照されない場合でも、操作によって検査することができます。

MemberDefinition`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE Type`Type of acceleration structure to build (see [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE](#d3d12_raytracing_acceleration_structure_type)).` D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS Flags``D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS ` to use for the build.`UINT NumDescs`<p> If Type is `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TOP_LEVEL`, number of instances (laid out based on DescsLayout).</p><p>If Type is `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BOTTOM_LEVEL`, number of elements pGeometryDescs or ppGeometryDescs refer to (which one is used depends on DescsLayout).</p>`D3D12_ELEMENTS_LAYOUT DescsLayout`How geometry descs are specified (see [D3D12_ELEMENTS_LAYOUT](#d3d12_elements_layout)): an array of descs or an array of pointers to descs.`const D3D12_GPU_VIRTUAL_ADDRESS InstanceDescs`<p> If Type is `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TOP_LEVEL`, this refers to NumDescs [D3D12_RAYTRACING_INSTANCE_DESC](#d3d12_raytracing_instance_desc) structures in GPU memory describing instances. Each instance must be aligned to 16 bytes ([D3D12_RAYTRACING_INSTANCE_DESC_BYTE_ALIGNMENT](#constants)).</p><p>If DescLayout is `D3D12_ELEMENTS_LAYOUT_ARRAY`, InstanceDescs points to an array of instance descs in GPU memory.</p><p>If DescLayout is `D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS`, InstanceDescs points to an array in GPU memory of `D3D12_GPU_VIRTUAL_ADDRESS` pointers to instance descs.</p><p>If Type is not `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TOP_LEVEL`, this parameter is unused (space repurposed in a union).</p><p>The memory pointed to must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`.</p>`const D3D12_RAYTRACING_GEOMETRY_DESC* pGeometryDescs`<p>If Type is `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BOTTOM_LEVEL`, and DescsLayout is `D3D12_ELEMENTS_LAYOUT_ARRAY`, this field is used and points to NumDescs contiguous [D3D12_RAYTRACING_GEOMETRY_DESC](#d3d12_raytracing_geometry_desc) structures on the CPU describing individual geometries.</p><p>If Type is not `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BOTTOM_LEVEL` or DescsLayout is not `D3D12_ELEMENTS_LAYOUT_ARRAY`, this parameter is unused (space repurposed in a union).</p><p>_The reason pGeometryDescs is a CPU based parameter as opposed to InstanceDescs which live on the GPU is, at least for initial implementations, the CPU needs to look at some of the information such as triangle counts in pGeometryDescs in order to schedule acceleration structure builds. Perhaps in the future more of the data can live on the GPU._</p>`const D3D12_RAYTRACING_GEOMETRY_DESC** ppGeometryDescs`<p>If Type is `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BOTTOM_LEVEL`, and DescsLayout is `D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS`, this field is used and points to an array of NumDescs pointers to [D3D12_RAYTRACING_GEOMETRY_DESC](#d3d12_raytracing_geometry_desc) structures on the CPU describing individual geometries.</p><p>If Type is not `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BOTTOM_LEVEL` or DescsLayout is not `D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS`, this parameter is unused (space repurposed in a union).</p><p>_ppGeometryDescs is a CPU based parameter for the same reason as pGeometryDescs described above. The only difference is this option lets the app have sparsely located geometry descs if desired._</p>---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE

```C++
typedef enum D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE
{
  D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_TOP_LEVEL = 0x0,
  D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL = 0x1
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE;
```

ValueDefinition`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_TOP_LEVEL`Top-level acceleration structure.`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL`Bottom-level acceleration structure.これらのタイプの説明は、ジオメトリと acceleration structure で、レイ-ジオメトリ相互作用図に視覚化されています。

---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS

```C++
typedef enum D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS
{
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_NONE = 0x00,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_UPDATE =0x01,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_COMPACTION= 0x02,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE= 0x04,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_BUILD= 0x08,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_MINIMIZE_MEMORY= 0x10,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PERFORM_UPDATE= 0x20,
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAGS;
```

MemberDefinition`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_NONE`No options specified for the acceleration structure build.`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_UPDATE`<p> Build the acceleration structure such that it supports future updates (via the flag `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PERFORM_UPDATE`) instead of the app having to entirely rebuild. This option may result in increased memory consumption, build times and lower raytracing performance. Future updates, however, should be faster than building the equivalent acceleration structure from scratch.</p><p> This flag can only be set on an initial acceleration structure build, or on an update where the source acceleration structure specified `ALLOW_UPDATE`. In other words as soon as an acceleration structure has been built without `ALLOW_UPDATE`, no other acceleration structures can be created from it via updates.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_COMPACTION`<p>Enables the option to compact the acceleration structure by calling [CopyRaytracingAccelerationStructure()](#copyraytracingaccelerationstructure) with the compact mode (see [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE](#d3d12_raytracing_acceleration_structure_copy_mode)).</p><p> This option may result in increased memory consumption and build times. After future compaction, however, the resulting acceleration structure should consume a smaller memory footprint (certainly no larger) than building the acceleration structure from scratch.</p><p>Specifying `ALLOW_COMPACTION` may increase pre-compaction acceleration structure size versus not specifying `ALLOW_COMPACTION`.</p><p>If multiple incremental builds are performed before finally compacting, there may be redundant compaction related work performed.</p><p>The size required for the compacted acceleration structure can be queried before compaction via [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo) -- see [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_compacted_size_desc) in particular for a discussion on some properties of compacted acceleration structure size.</p><p>This flag is compatible with all other flags. If specified as part of an acceleration structure update, the source acceleration structure must have also been built with this flag. In other words as soon as an acceleration structure has been built without `ALLOW_COMPACTION`, no other acceleration structures can be created from it via updates that specify `ALLOW_COMPACTION`.</p><p>_Note on interaction of `ALLOW_UPDATE` with `ALLOW_COMPACTION` that might apply to some implementations:_</p><p>_As long as `ALLOW_UPDATE` is specified, there is certain information that needs to be retained in the acceleration structure, and compaction will only help so much._</p><p>_However, if the implementation knows that the acceleration structure will no longer be updated, it could do a better job of compacting it._</p><p>_The application could benefit from compacting twice - once after the initial build, and once after the acceleration structure has "settled" to a static state (if ever)._</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE`<p>Construct a high quality acceleration structure that maximizes raytracing performance at the expense of additional build time. A rough rule of thumb is the implementation should take about 2-3 times the build time than default in order to get better tracing performance.</p><p> This flag is recommended for static geometry in particular. It is also compatible with all other flags except for `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_BUILD`.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_BUILD`<p>Construct a lower quality acceleration structure, trading raytracing performance for build speed. A rough rule of thumb is the implementation should take about 1/2 to 1/3 the build time than default at a sacrifice in tracing performance.</p><p>This flag is compatible with all other flags except for `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE`.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_MINIMIZE_MEMORY`<p>Minimize the amount of scratch memory used during the acceleration structure build as well as the size of the result. This option may result in increased build times and/or raytracing times.</p><p>This is orthogonal to the `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_COMPACTION` flag (and explicit acceleration structure compaction that it enables). Combining the flags can mean both the initial acceleration structure as well as the result of compacting it use less memory.</p><p>The impact of using this flag for a build is reflected in the result of calling [GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo) before doing the build to retrieve memory requirements for the build.</p><p>This flag is compatible with all other flags.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PERFORM_UPDATE`<p> Perform an acceleration structure update, as opposed to building from scratch. This is faster than a full build, but can negatively impact raytracing performance, especially if the positions of the underlying objects have changed significantly from the original build of the acceleration structure before updates.</p><p>See [Acceleration structure update constraints](#acceleration-structure-update-constraints) for a discussion of what is allowed to change in an acceleration structure update.</p><p>If the addresses of the source and destination acceleration structures are identical, the update is performed in-place. Any other overlapping of address ranges of the source and destination is invalid. For non-overlapping source and destinations, the source acceleration structure is unmodified. The memory requirement for the output acceleration structure is the same as in the input acceleration structure.</p><p>The source acceleration structure must have specified `ALLOW_UPDATE`.</p><p>This flag is compatible with all other flags. The other flags selections, aside from `ALLOW_UPDATE` and `PERFORM_UPDATE`, must match the flags in the source acceleration structure.</p><p>Acceleration structure updates can be performed in unlimited succession, as long as the source acceleration structure was created with `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_UPDATE` and the flags for the update build continue to specify `ALLOW_UPDATE`.---

##### D3D12_ELEMENTS_LAYOUT

```C++
typedef enum D3D12_ELEMENTS_LAYOUT
{
    D3D12_ELEMENTS_LAYOUT_ARRAY = 0x0,
    D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS = 0x1
} D3D12_ELEMENTS_LAYOUT;
```

n 個の要素からなるデータセットが与えられたとき、要素の位置がどのように特定されるかを記述する。

MemberDefinition`D3D12_ELEMENTS_LAYOUT_ARRAY`For a data set of n elements, the pointer parameter simply points to the start of an of n elements in memory.`D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS`For a data set of n elements, the pointer parameter points to an array of n pointers in memory, each pointing to an individual element of the set.---

##### D3D12_RAYTRACING_GEOMETRY_DESC

```C++
typedef struct D3D12_RAYTRACING_GEOMETRY_DESC
{
    D3D12_RAYTRACING_GEOMETRY_TYPE Type;
    D3D12_RAYTRACING_GEOMETRY_FLAGS Flags;
    union
    {
        D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC Triangles;
        D3D12_RAYTRACING_GEOMETRY_AABBS_DESC AABBs;
    };
} D3D12_RAYTRACING_GEOMETRY_DESC;
```

MemberDefinition`D3D12_RAYTRACING_GEOMETRY_TYPE Type`[D3D12_RAYTRACING_GEOMETRY_TYPE](#d3d12_raytracing_geometry_type) for this geometry.`D3D12_RAYTRACING_GEOMETRY_FLAGS Flags`[D3D12_RAYTRACING_GEOMETRY_FLAGS](#d3d12_raytracing_geometry_flags) for this geometry.`D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC Triangles`[D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc) describing triangle geometry if Type is `D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES`. Otherwise this parameter is unused (space repurposed in a union).`D3D12_RAYTRACING_GEOMETRY_AABBS_DESC AABBs`[D3D12_RAYTRACING_GEOMETRY_AABBS_DESC](#d3d12_raytracing_geometry_aabbs_desc) describing AABB geometry if Type is `D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS`. Otherwise this parameter is unused (space repurposed in a union).---

##### D3D12_RAYTRACING_GEOMETRY_TYPE

```C++
typedef enum D3D12_RAYTRACING_GEOMETRY_TYPE
{
    D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES,
    D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS
} D3D12_RAYTRACING_GEOMETRY_TYPE;
```

ValueDefinition`D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES`The geometry consists of triangles described by [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc).`D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS`The geometry procedurally is defined during raytracing by [intersection shaders](#intersection-shaders---procedural-primitive-geometry). So for the purpose of acceleration structure builds, the geometry's bounds are described with axis-aligned bounding boxes via [D3D12_RAYTRACING_GEOMETRY_AABBS_DESC](#d3d12_raytracing_geometry_aabbs_desc).---

##### D3D12_RAYTRACING_GEOMETRY_FLAGS

```C++
typedef enum D3D12_RAYTRACING_GEOMETRY_FLAGS
{
    D3D12_RAYTRACING_GEOMETRY_FLAG_NONE = 0x0,
    D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE = 0x1,
    D3D12_RAYTRACING_GEOMETRY_FLAG_NO_DUPLICATE_ANYHIT_INVOCATION = 0x2,
} D3D12_RAYTRACING_GEOMETRY_FLAGS;
```

ValueDefinition`D3D12_RAYTRACING_GEOMETRY_FLAG_NONE`No options specified.`D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE`When rays encounter this geometry, the geometry acts as if no any hit shader is present. It is recommended to use this flag liberally, as it can enable important ray processing optimizations. Note that this behavior can be overridden on a per-instance basis with [D3D12_RAYTRACING_INSTANCE_FLAGS](#d3d12_raytracing_instance_flags) and on a per-ray basis using [Ray flags](#ray-flags) in [TraceRay()](#traceray) and [RayQuery::TraceRayInline()](#rayquery-tracerayinline).`D3D12_RAYTRACING_FLAG_NO_DUPLICATE_ANYHIT_INVOCATION`<p>By default, the system is free to trigger an [any hit shader](#any-hit-shaders) more than once for a given ray-primitive intersection. This flexibility helps improve the traversal efficiency of acceleration structures in certain cases. For instance, if the acceleration structure is implemented internally with bounding volumes, the implementation may find it beneficial to store relatively long triangles in multiple bounding boxes rather than a larger single box.</p><p>However, some application use cases require that intersections be reported to the any hit shader at most once. This flag enables that guarantee for the given geometry, potentially with some performance impact.</p><p>This flag applies to all [geometry types](#d3d12_raytracing_geometry_type).</p>---

##### D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC

```C++
typedef struct D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC
{
    D3D12_GPU_VIRTUAL_ADDRESS Transform3x4;
    DXGI_FORMAT IndexFormat;
    DXGI_FORMAT VertexFormat;
    UINT IndexCount;
    UINT VertexCount;
    D3D12_GPU_VIRTUAL_ADDRESS IndexBuffer;
    D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE VertexBuffer;
} D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC;
```

この構造体が指すジオメトリは、常に三角形のリスト（インデックス付きまたはインデックスなしの形式）です。簡略化のため、ストリップはサポートされていません。

MemberDefinition`D3D12_GPU_VIRTUAL_ADDRESS Transform3x4`<p>Address of a 3x4 affine transform matrix in row major layout to be applied to the vertices in the VertexBuffer during an acceleration structure build. The contents of VertexBuffer are not modified. If a 2D vertex format is used, the transformation is applied with the third vertex component assumed to be zero.</p><p>If Transform is NULL the vertices will not be transformed. Using Transform may result in increased computation and/or memory requirements for the acceleration structure build.</p><p>The memory pointed to must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. The address must be aligned to 16 bytes ([D3D12_RAYTRACING_TRANSFORM3X4_BYTE_ALIGNMENT](#constants)).</p>`DXGI_FORMAT IndexFormat`<p>Format of the indices in the IndexBuffer. Must be one of:</p><p>`DXGI_FORMAT_UNKNOWN` (when IndexBuffer is NULL)</p><p>`DXGI_FORMAT_R32_UINT`</p><p>`DXGI_FORMAT_R16_UINT`</p>`DXGI_FORMAT VertexFormat`<p>Format of the vertices (positions) in VertexBuffer. Must be one of:</p><p>`DXGI_FORMAT_R32G32_FLOAT` (third component assumed 0)</p><p>`DXGI_FORMAT_R32G32B32_FLOAT`</p><p>`DXGI_FORMAT_R16G16_FLOAT` (third component assumed 0)</p><p>`DXGI_FORMAT_R16G16B16A16_FLOAT` (A16 component is ignored, other data can be packed there, such as setting vertex stride to 6 bytes)</p><p>`DXGI_FORMAT_R16G16_SNORM` (third component assumed 0)</p><p>`DXGI_FORMAT_R16G16B16A16_SNORM` (A16 component is ignored, other data can be packed there, such as setting vertex stride to 6 bytes)</p><p>[Tier 1.1](#d3d12_raytracing_tier) devices support the following additional formats:</p><p>`DXGI_FORMAT_R16G16B16A16_UNORM` (A16 component is ignored, other data can be packed there, such as setting vertex stride to 6 bytes)</p><p>`DXGI_FORMAT_R16G16_UNORM` (third component assumed 0)</p><p>`DXGI_FORMAT_R10G10B10A2_UNORM` (A2 component is ignored, stride must be 4 bytes)</p><p>`DXGI_FORMAT_R8G8B8A8_UNORM` (A8 component is ignored, other data can be packed there, such as setting vertex stride to 3 bytes)</p><p>`DXGI_FORMAT_R8G8_UNORM` (third component assumed 0)</p><p>`DXGI_FORMAT_R8G8B8A8_SNORM` (A8 component is ignored, other data can be packed there, such as setting vertex stride to 3 bytes)</p><p>`DXGI_FORMAT_R8G8_SNORM` (third component assumed 0)</p><p>_The 4 component formats with ignored A\* components were a way around having to introduce 3 component variants of these formats to the DXGI format list (and associated infrastructure) only for this scenario. So this just saved a minor mount of engineering work given too much work to do overall._</p>`UINT IndexCount`Number of indices in IndexBuffer. Must be 0 if IndexBuffer is NULL.`UINT VertexCount`Number of vertices (positions) in VertexBuffer. If an index buffer is present, this must be at least the maximum index value in the index buffer + 1.`D3D12_GPU_VIRTUAL_ADDRESS IndexBuffer`<p>Array of vertex indices. If NULL, triangles are non-indexed. Just as with graphics, the address must be aligned to the size of IndexFormat.</p><p>The memory pointed to must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Note that if an app wants to share index buffer inputs between graphics input assembler and raytracing acceleration structure build input, it can always put a resource into a combination of read states simultaneously, e.g. `D3D12_RESOURCE_STATE_INDEX_BUFFER | D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`.</p>`D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE VertexBuffer`<p> Array of vertices including a stride. The alignment on the address and stride must be a multiple of the component size, so 4 bytes for formats with 32bit components and 2 bytes for formats with 16bit components. There is no constraint on the stride (whereas there is a limit for graphics), other than that the bottom 32bits of the value are all that are used -- the field is UINT64 purely to make neighboring fields align cleanly/obviously everywhere. Each vertex position is expected to be at the start address of the stride range and any excess space is ignored by acceleration structure builds. This excess space might contain other app data such as vertex attributes, which the app is responsible for manually fetching in shaders, whether it is interleaved in vertex buffers or elsewhere.</p><p>The memory pointed to must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Note that if an app wants to share vertex buffer inputs between graphics input assembler and raytracing acceleration structure build input, it can always put a resource into a combination of read states simultaneously, e.g. `D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER | D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`.</p>---

##### D3D12_RAYTRACING_GEOMETRY_AABBS_DESC

```C++
typedef struct D3D12_RAYTRACING_GEOMETRY_AABBS_DESC
{
    UINT64 AABBCount;
    D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE AABBs;
} D3D12_RAYTRACING_GEOMETRY_AABBS_DESC;
```

MemberDefinition`UINT AABBCount`Number of AABBs pointed to in the contiguous array at AABBs.`D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE AABBs`<p>[D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE](#d3d12_gpu_virtual_address_and_stride) describing the GPU memory location where an array of [AABB descriptions](#d3d12_raytracing_aabb) is to be found, including the data stride between AABBs. The address and stride must each be aligned to 8 bytes ([D3D12_RAYTRACING_AABB_BYTE_ALIGNMENT](#constants)).</p><p>The stride can be 0.</p><p>The memory pointed to must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`.</p>---

##### D3D12_RAYTRACING_AABB

```C++
typedef struct D3D12_RAYTRACING_AABB
{
    FLOAT MinX;
    FLOAT MinY;
    FLOAT MinZ;
    FLOAT MaxX;
    FLOAT MaxY;
    FLOAT MaxZ;
} D3D12_RAYTRACING_AABB;
```

MemberDefinition`FLOAT MinX, MinY, MinZ`The minimum X, Y and Z coordinates of the box.`FLOAT MaxX, MaxY, MaxZ`The maximum X, Y and Z coordinates of the box.---

##### D3D12_RAYTRACING_INSTANCE_DESC

```C++
typedef struct D3D12_RAYTRACING_INSTANCE_DESC
{
    FLOAT Transform[3][4];
    UINT InstanceID : 24;
    UINT InstanceMask : 8;
    UINT InstanceContributionToHitGroupIndex : 24;
    UINT Flags : 8;
    D3D12_GPU_VIRTUAL_ADDRESS AccelerationStructure;
} D3D12_RAYTRACING_INSTANCE_DESC;
```

このデータ構造は、アクセラレーション構造構築時に GPU メモリで使用されます。この C++ 構造体の定義は、まず CPU でインスタンスデータを生成し、それから GPU にアップロードする場合に便利です。しかし、アプリは同じレイアウトに従って、たとえばコンピュートシェーダから GPU メモリに直接インスタンス記述を生成することも自由です。

MemberDefinition`FLOAT Transform[3][4]`A 3x4 transform matrix in row major layout representing the instance-to-world transformation. Implementations transform rays, as opposed to transforming all the geometry/AABBs.`UINT InstanceID`An arbitrary 24-bit value that can be accessed via [InstanceID()](#instanceid) in shader types listed in [System value intrinsics](#system-value-intrinsics).`UINT InstanceMask`An 8-bit mask assigned to the instance, which can be used to include/reject groups of instances on a per-ray basis. See the InstanceInclusionMask parameter in [TraceRay()](#traceray) and [RayQuery::TraceRayInline()](#rayquery-tracerayinline). If the value is zero, the instance will never be included, so typically this should be set to some nonzero value.`UINT InstanceContributionToHitGroupIndex`Per-instance contribution to add into shader table indexing to select the hit group to use. The indexing behavior is introduced here: [Indexing into shader tables](#indexing-into-shader-tables), detailed here: [Addressing calculations within shader tables](#addressing-calculations-within-shader-tables), and visualized here: [Ray-geometry interaction diagram](#ray-geometry-interaction-diagram). Has no behavior with [inline raytracing](#inline-raytracing), however the value is still available to fetch from a [RayQuery](#rayquery) object for shaders use manually for any purpose.`UINT Flags`Flags from [D3D12_RAYTRACING_INSTANCE_FLAGS](#d3d12_raytracing_instance_flags) to apply to the instance.`D3D12_GPU_VIRTUAL_ADDRESS AccelerationStructure`<p>Address of the bottom-level acceleration structure that is being instanced. The address must be aligned to 256 bytes ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)), which is a somewhat redundant requirement as any existing acceleration structure passed in here would have already been required to be placed with such alignment anyway.</p><p>The memory pointed to must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>---

##### D3D12_RAYTRACING_INSTANCE_FLAGS

```C++
typedef enum D3D12_RAYTRACING_INSTANCE_FLAGS
{
    D3D12_RAYTRACING_INSTANCE_FLAG_NONE = 0x0,
    D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_CULL_DISABLE = 0x1,
    D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE =0x2,
    D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_OPAQUE = 0x4,
    D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_NON_OPAQUE = 0x8
} D3D12_RAYTRACING_INSTANCE_FLAGS;
```

ValueDefinition`D3D12_RAYTRACING_INSTANCE_FLAG_NONE`No options specified.`D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_CULL_DISABLE`Disables front/back face culling for this instance. The [Ray flags](#ray-flags) `RAY_FLAG_CULL_BACK_FACING_TRIANGLES` and `RAY_FLAG_CULL_FRONT_FACING_TRIANGLES` will have no effect on this instance. `RAY_FLAG_SKIP_TRIANGLES` takes precedence however - if that flag is set, then `D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_CULL_DISABLE` has no effect, and all triangles will be skipped.`D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE`<p> This flag reverses front and back facings, which is useful if for example, the application's natural winding order differs from the default (described below).</p><p>By default, a triangle is front facing if its vertices appear clockwise from the ray origin and back facing if its vertices appear counter-clockwise from the ray origin, in object space in a left-handed coordinate system.</p><p>Since these winding direction rules are defined in object space, they are unaffected by instance transforms. For example, an instance transform matrix with negative determinant (e.g. mirroring some geometry) does not change the facing of the triangles within the instance. Per-geometry transforms, by contrast, (defined in [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc)), get combined with the associated vertex data in object space, so a negative determinant matrix there _does_ flip triangle winding.</p>`D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_OPAQUE`<p>The instance will act as if [D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE](#d3d12_raytracing_geometry_flags) had been specified for all the geometries in the bottom-level acceleration structure referenced by the instance. Note that this behavior can be overridden by the [ray flag](#ray-flags) `RAY_FLAG_FORCE_NON_OPAQUE`.</p><p>Mutually exclusive to the `D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_NON_OPAQUE` flag.</p>`D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_NON_OPAQUE`<p>The instance will act as if [D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE](#d3d12_raytracing_geometry_flags) had not been specified for any of the geometries in the bottom-level acceleration structure referenced by the instance. Note that this behavior can be overridden by the [ray flag](#ray-flags) `RAY_FLAG_FORCE_OPAQUE`.</p><p>Mutually exclusive to the `D3D12_RAYTRACING_INSTANCE_FLAG_FORCE_OPAQUE` flag.</p>---

##### D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE

```C++
typedef struct D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE
{
    D3D12_GPU_VIRTUAL_ADDRESS StartAddress;
    UINT64 StrideInBytes;
} D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE;
```

MemberDefinition`UINT64 StartAddress`Beginning of a VA range.`UINT64 StrideInBytes`Defines indexing stride, such as for vertices. Only the bottom 32 bits get used. The field is 64 bits purely to make alignment of containing structures clean/obvious everywhere.---

### EmitRaytracingAccelerationStructurePostbuildInfo

```C++
void EmitRaytracingAccelerationStructurePostbuildInfo(
    _In_ const D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC* pDesc,
    _In_ UINT NumSourceAccelerationStructures,
    _In_reads_( NumSourceAccelerationStructures ) const D3D12_GPU_VIRTUAL_ADDRESS*
              pSourceAccelerationStructureData);
```

アクセラレーション構造体のセットに対するポストビルドプロパティを出力します。これにより、アプリケーションは CopyRaytracingAccelerationStructure() によるアクセラレーション構造の操作を実行する際の出力リソースの要件を知ることができ ます。

graphics や compute コマンドリストで呼び出せますが、bundle からは呼び出せません。

ParameterDefinition`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC* pDesc`Description of postbuild information to generate.`UINT NumSourceAccelerationStructures`Number of pointers to acceleration structure GPUVAs pointed to by pSourceAccelerationStructureData. This number also affects the destination (output), which will be a contiguous array of NumSourceAccelerationStructures output structures, where the type of the structures depends on InfoType.`const D3D12_GPU_VIRTUAL_ADDRESS* pSourceAccelerationStructureData`<p>Pointer to array of GPUVAs of size NumSourceAccelerationStructures. Each GPUVA points to the start of an existing acceleration structure, which is aligned to 256 bytes ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)).</p><p>The memory pointed to must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>---

#### EmitRaytracingAccelerationStructurePostbuildInfo Structures

---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC
{
    D3D12_GPU_VIRTUAL_ADDRESS DestBuffer;
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TYPE InfoType;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC;
```

acceleration structure から生成するポストビルド情報の説明。これは EmitRaytracingAccelerationStructurePostbuildInfo() とオプションで BuildRaytracingAccelerationStructure() の両方で使用されます。

MemberDefinition`D3D12_GPU_VIRTUAL_ADDRESS DestBuffer`<p>Result storage. Size required and the layout of the contents written by the system depend on InfoType.</p><p>The memory pointed to must be in state `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`. The memory must be aligned to the natural alignment for the members of the particular output structure being generated (e.g. 8 bytes for a struct with the largest membes being UINT64).</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TYPE InfoType`Type of postbuild information to retrieve.---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TYPE

```C++
typedef enum D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TYPE
{
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE = 0x0,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION = 0x1,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION = 0x2,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE = 0x3,
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TYPE;
```

MemberDefinition`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE`Space requirements for an acceleration structure after compaction. See [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_compacted_size_desc).`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION`Space requirements for generating tools visualization for an acceleration structure (used by tools). See [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_tools_visualization_desc).`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION`Space requirements for serializing an acceleration structure. See [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_serialization_desc)`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE`Size of the current acceleration structure. See [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_current_size_desc).---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE_DESC

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE_DESC
{
    UINT64 CompactedSizeInBytes;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE_DESC;
```

MemberDefinition`UINT64 CompactedSizeInBytes`<p>Space requirement for acceleration structure after compaction.</p><p>It is guaranteed that a compacted acceleration structure doesn't consume more space than a non-compacted acceleration structure.</p><p>Pre-compaction, it is guaranteed that the size requirements reported by [GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo) for a given build configuration (triangle counts etc.) will be sufficient to store any acceleration structure whose build inputs are reduced (e.g. reducing triangle counts). This is discussed in [GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo). This non-increasing property for smaller builds does not apply post-compaction, however. In other words, it is not guaranteed that having fewer items in an acceleration structure means it compresses to a smaller size than compressing an acceleration structure with more items.</p>---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION_DESC

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION_DESC
{
    UINT64 DecodedSizeInBytes;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_TOOLS_VISUALIZATION_DESC;
```

MemberDefinition`UINT64 DecodedSizeInBytes`Space requirement for decoding an acceleration structure into a form that can be visualized by tools.---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC
{
    UINT64 SerializedSizeInBytes;
    UINT64 NumBottomLevelAccelerationStructurePointers; // UINT64 to align arrays of this struct
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC;
```

MemberDefinition`UINT64 SerializedSizeInBytes | Size of the serialized acceleration structure, including a header. The header is [D3D12_SERIALIZED_ACCELERATION_STRUCTURE_HEADER](#d3d12_serialized_acceleration_structure_header) followed by followed by a list of pointers to bottom-level acceleration structures.`UINT64 NumBottomLevelAccelerationStructurePointers`| <p>How many 64bit GPUVAs will be at the start of the serialized acceleration structure (after`D3D12_SERIALIZED_ACCELERATION_STRUCTURE_HEADER` above). For a bottom-level acceleration structure this will be 0. For a top-level acceleration structure, the pointers indicate the acceleration structures being referred to.</p><p> When deserializing happens, these pointers to bottom level pointers must be initialized by the app in the serialized data (just after the header) to the new locations where the bottom level acceleration structures will reside. These new locations pointed to at deserialize time need not have been populated with bottom-level acceleration structures yet, as long as they have been initialized with the expected deserialized data structures before use in raytracing. During deserialization, the driver reads the new pointers, using them to produce an equivalent top-level acceleration structure to the original.</p>---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE_DESC

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE_DESC
{
    UINT64 CurrentSizeInBytes;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_CURRENT_SIZE_DESC;
```

MemberDefinition`UINT64 CurrentSizeInBytes**`<p>Space used by current acceleration structure. If the acceleration structure hasn't had a compaction operation performed on it, this size is the same one reported by [GetRaytracingAccelerationStructurePrebuildInfo()](#getraytracingaccelerationstructureprebuildinfo), and if it has been compacted this size is the same reported for postbuild info with `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE`.</p><p>_While this appears redundant to other ways of querying sizes, this one can be handy for tools to be able to determine how much memory is occupied by an arbitrary acceleration structure sitting in memory._</p>---

### CopyRaytracingAccelerationStructure

```C++
void CopyRaytracingAccelerationStructure(
    _In_ D3D12_GPU_VIRTUAL_ADDRESS DestAccelerationStructureData,
    _In_ D3D12_GPU_VIRTUAL_ADDRESS SourceAccelerationStructureData,
    _In_ D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE Mode);
```

レイトレーシング アクセラレーション構造体は内部ポインターを含み、デバイスに依存した不透明なレイアウトを持つことがあるので、それらをコピーしたり、そうでなければ操作するには、ドライバーが要求された操作を処理できるように、専用の API が必要です。この API は、Mode パラメータで要求された変換を適用しながら、ソース acceleration structure を受け取り、それをデスティネーションメモリにコピーします。

graphics や compute コマンドリストで呼び出せますが、bundle からは呼び出せません。

ParameterDefinition`D3D12_GPU_VIRTUAL_ADDRESS DestAccelerationStructureData`<p>Destination memory. Required size can be discovered by calling [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo) beforehand, if necessary depending on the Mode.</p><p> Destination address must be 256 byte aligned ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)), regardless of the Mode.</p><p> Destination memory range cannot overlap source otherwise results are undefined.</p><p>The resource state that the memory pointed to must be in depends on the Mode parameter - see [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE](#d3d12_raytracing_acceleration_structure_copy_mode) definitions.</p>`D3D12_GPU_VIRTUAL_ADDRESS SourceAccelerationStructureData`<p> Acceleration structure or other type of data to copy/transform based on the specified Mode. The data remains unchanged and usable (such as if it is an acceleration structure). The operation only involves the data pointed to (such as acceleration structure) and not any other data (such as acceleration structures) it may point to. E.g. in the case of a top-level acceleration structure, any bottom-level acceleration structures that it points to are not involved in the operation.</p><p>Memory must be 256 byte aligned ([D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT](#constants)) regardless of Mode.</p><p>The resource state that the memory pointed to must be in depends on the Mode parameter - see [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE](#d3d12_raytracing_acceleration_structure_copy_mode) definitions.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE Mode`Type of copy operation to perform. See [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE](#d3d12_raytracing_acceleration_structure_copy_mode).---

#### CopyRaytracingAccelerationStructure Structures

---

##### D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE

```C++
typedef enum D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE
{
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_CLONE = 0x0,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_COMPACT = 0x1,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_VISUALIZATION_DECODE_FOR_TOOLS = 0x2,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_SERIALIZE = 0x3,
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_DESERIALIZE = 0x4,
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE;
```

ValueDefinition`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_CLONE`<p>Copy an acceleration structure while fixing up any self-referential pointers that may be present so that the destination is a self-contained match for the source. Any external pointers to other acceleration structures remain unchanged from source to destination in the copy. The size of the destination is identical to the size of the source.</p><p>The source and destination memory must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_COMPACT`<p>Similar to the clone mode, producing a functionally equivalent acceleration structure to source in the destination. Compact mode also fits the destination into a potentially smaller memory footprint (certainly no larger). The size required for the destination can be retrieved beforehand from [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo).</p><p>This mode is only valid if the source acceleration structure was originally built with the [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_COMPACTION](#d3d12_raytracing_acceleration_structure_build_flags) flag, otherwise results are undefined.</p><p>The source and destination memory must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_VISUALIZATION_DECODE_FOR_TOOLS`<p>Destination takes the layout described in [D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_TOOLS_VISUALIZATION_HEADER](#d3d12_build_raytracing_acceleration_structure_tools_visualization_header). The size required for the destination can be retrieved beforehand from [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo).</p><p>This mode is intended for tools such as PIX only, though nothing stops any app from using it. The output is essentially the inverse of an acceleration structure build and is self-contained in the destination buffer.</p><p>For top-level acceleration structures, the output includes a set of instance descriptions that are identical to the data used in the original build and in the same order.</p><p>For bottom-level acceleration structures, the output includes a set of geometry descriptions _roughly_ matching the data used in the original build. The output is only a rough match for the original in part because of the tolerances allowed in the specification for [acceleration structures](#acceleration-structure-properties), and in part because reporting exactly the same structure as is conceptually encoded may not be simple.</p><p>AABBs returned for procedural primitives, for instance, could be more conservative (larger) in volume and even different in number than what is actually in the acceleration structure representation (because it may not be clean to expose the exact representation).</p><p>Geometries (each with its own geometry description) must appear in the same order as in the original build, as [shader table indexing](#hit-group-table-indexing) calculations depends on this.</p><p> This overall structure with is sufficient for tools/PIX to be able to give the application some visual sense of the acceleration structure the driver made out of the app's input. Visualization can help reveal driver bugs in acceleration structures if what is shown grossly mismatches the data the application used to create the acceleration structure, beyond allowed tolerances.</p><p>The source memory must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p><p>The destination memory must be in state `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`.</p><p>This mode is only permitted when developer mode is enabled on the OS.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_SERIALIZE`<p>Destination takes the layout and size described in the documentation for [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_serialization_desc), itself a struct generated via [EmitRaytracingAccelerationStructurePostbuildInfo()](#emitraytracingaccelerationstructurepostbuildinfo).</p><p>This mode serializes an acceleration structure so that an app or tools/PIX can store it to a file for later reuse, typically on a different device instance, via deserialization.</p><p>The source memory must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p><p>The destination memory must be in state `D3D12_RESOURCE_STATE_UNORDERED_ACCESS`.</p><p>When serializing a top-level acceleration structure the bottom-level acceleration structures it refers to do not have to still be present/intact in memory. Likewise bottom-level acceleration structures can be serialized independent of whether any top-level acceleration structures are pointing to them. Said another way, order of serialization of acceleration structures doesn't matter.</p>`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_DESERIALIZE`<p>Source must be a serialized acceleration structure, with any pointers (directly after the header) fixed to point to their new locations, as discussed in the [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_serialization_desc) section.</p><p>Destination gets an acceleration structure that is functionally equivalent to the acceleration structure that was originally serialized. It does not matter what order top-level and bottom-level acceleration structures (that the top-level refers to) are deserialized, as long as by the time a top-level acceleration structure is used for raytracing or acceleration structure updates it's referenced bottom-level acceleration structures are present.</p><p>Deserialize only works on the same device and driver version otherwise results are undefined. This isn't intended to be used for caching acceleration structures, as running a full acceleration structure build is likely to be faster than loading one from disk.</p><p>While intended for tools/PIX, nothing stops any app from using this, though at least for now deserialization requires the OS to be in developer mode.</p><p>The source memory must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`.</p><p>The destination memory must be in state [D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE](#additional-resource-states).</p>---

##### D3D12_SERIALIZED_ACCELERATION_STRUCTURE_HEADER

```C++
typedef struct D3D12_SERIALIZED_RAYTRACING_ACCELERATION_STRUCTURE_HEADER
{
    D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER DriverMatchingIdentifier;
    UINT64 SerializedSizeInBytesIncludingHeader;
    UINT64 DeserializedSizeInBytes;
    UINT64 NumBottomLevelAccelerationStructurePointersAfterHeader;
} D3D12_SERIALIZED_RAYTRACING_ACCELERATION_STRUCTURE_HEADER;
```

シリアライズされたレイトレーシングのアクセラレーション構造体のヘッダーです。

MemberDefinition`D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER DriverMatchingIdentifier`See [D3D12_SERIALIZED_DATA_DRIVER_MATCHING_IDENTIFIER](#d3d12_serialized_data_driver_matching_identifier).`UINT64 SerializedSizeInBytesIncludingHeader`Size of serialized data.`UINT64 DeserializedSizeInBytes`Size that will be consumed when deserialized. This is less than or equal to the size of the original acceleration structure before it was serialized.`UINT64 NumBottomLevelAccelerationStructurePointersAfterHeader`Size of the array of `D3D12_GPU_VIRTUAL_ADDRESS` values that follow the header. See discussion in [D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION_DESC](#d3d12_raytracing_acceleration_structure_postbuild_info_serialization_desc), which also reports the same count and explains what to do with these pointers.---

##### D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_TOOLS_VISUALIZATION_HEADER

```C++
typedef struct D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_TOOLS_VISUALIZATION_HEADER
{
    D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE Type;
    UINT NumDescs;
} D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_TOOLS_VISUALIZATION_HEADER;

// Depending on Type field, NumDescs above is followed by either:
// D3D12_RAYTRACING_INSTANCE_DESC InstanceDescs[NumDescs]
// or D3D12_RAYTRACING_GEOMETRY_DESC GeometryDescs[NumDescs].
```

これは、加速構造の視覚化の GPU メモリレイアウトを記述します。これは、アクセラレーション構造構築への入力の逆のようなもので、アクセラレーション構造のタイプに応じて、単にインスタンスまたはジオメトリの詳細に焦点を当てたものです。

---

### SetPipelineState1

```C++
void SetPipelineState1(_In_ ID3D12StateObject* pStateObject);
```

コマンドリスト上の状態オブジェクトを設定します。グラフィックスまたは計算コマンドリストおよびバンドルから呼び出すことができます。

これは、グラフィックスと計算シェーダにのみ定義されている SetPipelineState(In ID3D12PipelineState\*) の代わりとなるものです。一度にコマンドリスト上でアクティブになるパイプライン・ステートは 1 つだけなので、どちらの呼び出しも現在のパイプライン・ステートを設定します。この 2 つの呼び出しの違いは、それぞれが特定のタイプのパイプライン・ステートのみを設定することです。今のところ、少なくとも、SetPipelineState1 はレイトレーシング パイプラインの状態を設定するためだけのものです。

ParameterDefinition`ID3D12StateObject* pStateObject`State object. For now this can only be of [type](#d3d12_state_object_type): `D3D12_STATE_OBJECT_TYPE_RAYTRACING_PIPELINE`.---

### DispatchRays

```C++
void DispatchRays(_In_ const D3D12_DISPATCH_RAYS_DESC* pDesc);
```

レイ生成シェーダのスレッドを起動します。概要については、Initiating raytracing を参照してください。graphics または compute コマンドリストおよびバンドルから呼び出すことができます。

レイトレーシングのパイプラインの状態は、コマンドリストで設定されなければなりません、さもなければ、この呼び出しの動作は未定義です。

グリッドサイズを設定するために、3 つの次元が渡されます：幅/高さ/深さ。これらの寸法は、width*height*depth <= 2^30 となるように制約されます。これを超えると、未定義の動作になる。

いずれかのグリッド寸法が 0 の場合、スレッドは起動されません。

[Tier 1.1](#d3d12_raytracing_tier) implementations also support GPU initiated DispatchRays() via [ExecuteIndirect()](#executeindirect).

ParameterDefinition`const D3D12_DISPATCH_RAYS_DESC* pDesc`Description of the ray dispatch. See [D3D12_DISPATCH_RAYS_DESC](#d3d12_dispatch_rays_desc).---

#### DispatchRays Structures

---

##### D3D12_DISPATCH_RAYS_DESC

```C++
typedef struct D3D12_DISPATCH_RAYS_DESC
{
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE RayGenerationShaderRecord;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE MissShaderTable;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE HitGroupTable;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE CallableShaderTable;
    UINT Width;
    UINT Height;
    UINT Depth;
} D3D12_DISPATCH_RAYS_DESC;
```

MemberDefinition`D3D12_GPU_VIRTUAL_ADDRESS_RANGE RayGenerationShaderRecord`[Shader record](#shader-record) for the [Ray generation shader](#ray-generation-shaders) to use. Memory must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Address must be aligned to 64 bytes ([D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT](#constants)).`D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE MissShaderTable`<p>[Shader table](#shader-tables) for [Miss shader](#miss-shaders). The stride is record stride, and must be aligned to 32 bytes ([D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT](#constants)), and in the range [0…4096] bytes.</p><p>Memory must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Address must be aligned to 64 bytes([D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT](#constants)).</p>`D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE HitGroupTable`<p>[Shader table](#shader-tables) for [Hit Group](#hit-groups). The stride is record stride, and must be aligned to 32 bytes ([D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT](#constants)), and in the range [0…4096] bytes.</p><p> Memory must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Address must be aligned to 64 bytes ([D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT](#constants)).</p>`D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE CallableShaderTable`<p> [Shader table](#shader-tables) for [Callable shaders](#callable-shaders). The stride is record stride, and must be aligned to 32 bytes ([D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT](#constants)), and in the range [0…4096] bytes.</p><p>Memory must be in state `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Address must be aligned to 64 bytes ([D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT](#constants)).</p>`UINT Width`Width of ray generation shader thread grid.`UINT Height`Height of ray generation shader thread grid.`UINT Depth`Depth of ray generation shader thread grid.---

##### D3D12_GPU_VIRTUAL_ADDRESS_RANGE

```C++
typedef struct D3D12_GPU_VIRTUAL_ADDRESS_RANGE
{
    D3D12_GPU_VIRTUAL_ADDRESS StartAddress;
    UINT64 SizeInBytes;
} D3D12_GPU_VIRTUAL_ADDRESS_RANGE;
```

MemberDefinition`UINT64 StartAddress`Beginning of a VA range.`UINT64 SizeInBytes`Size of a VA range.---

##### D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE

```C++
typedef struct D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE
{
    D3D12_GPU_VIRTUAL_ADDRESS StartAddress;
    UINT64 SizeInBytes;
    UINT64 StrideInBytes;
} D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE;
```

MemberDefinition`UINT64 StartAddress`Beginning of a VA range.`UINT64 SizeInBytes`Size of a VA range.`UINT64 StrideInBytes`Define record indexing stride within the memory range.---

### ExecuteIndirect

```C++
void ID3D12CommandList::ExecuteIndirect(
    ID3D12CommandSignature* pCommandSignature,
    UINT MaxCommandCount,
    ID3D12Resource* pArgumentBuffer,
    UINT64 ArgumentBufferOffset,
    ID3D12Resource* pCountBuffer,
    UINT64 CountBufferOffset
);
```

これはレイトレーシングに特化した API ではなく、単に間接的なコマンド実行を発行するための一般的な D3D API です。

レイトレーシングに関連するのは、 `ID3D12CommandSignature` が DispatchRays() への間接的な呼び出しをサポートするように構成できることです - CreateCommandSignature() を参照してください。

`DispatchRays` 用に設定されたコマンドシグネチャが `ExecuteIndirect` に渡されると、レイトレーシング パイプライン ステートがコマンドリストに現在設定されていなければならず、さもなければ動作は不定となります。

アプリケーションが ExecuteIndirect() 呼び出しからシェーダーテーブルの内容を変更することは、その ExecuteIndirect() 呼び出しが同じシェーダーテーブルを参照する場合（例えば DispatchRays() を介して）、無効とします。

---

## ID3D12StateObjectProperties methods

ID3D12StateObjectProperties は、ID3D12StateObject によってエクスポートされるインターフェイスです。以下のメソッドが ID3D12StateObjectProperties から公開されています。

---

### GetShaderIdentifier

```C++
const void* GetShaderIdentifier(LPWSTR pExportName);
```

シェーダーレコードで使用できるシェーダーの一意な識別子を取得する。これは、レイ生成シェーダー、Hit グループ、Miss シェーダー、Callable シェーダーに対してのみ有効です。状態オブジェクトはコレクションまたはレイトレーシング パイプライン・ステート・オブジェクトであることができ、シェーダは未解決の参照または必要なサブオブジェクトへの見つからない関連付けを持ってはなりません（言い換えれば、ドライバによって完全にコンパイル可能である）、さもなければ nullptr が返されます。

ParameterDefinition`LPWSTR pExportName`Entrypoint in the state object for which to retrieve an identifier.`Return value: const void*`<p>Returns a pointer to the shader identifier.</p><p>The data pointed to is valid as long as the state object it came from is valid. The size of the data returned is [D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES](#constants). Applications should copy and cache this data to avoid the cost of searching for it in the state object if it will need to be retrieved many times. The place the identifier actually gets used is in shader records within shader tables in GPU memory, which it is up to the app to populate.</p><p>The data itself globally identifies the shader, so even if the shader appears in a different state object (with same associations like any root signatures) it will have the same identifier.</p><p>If the shader isn't fully resolved in the state object, the return value is nullptr.</p>---

### GetShaderStackSize

```C++
UINT64 GetShaderStackSize(LPCWSTR pExportName);
```

HLSL でレイトレーシングのシェーダを呼び出すのに必要なスタックメモリの量を取得します。レイ生成シェーダでさえ、スタックの最下部にあるにもかかわらず、非ゼロを返すことがあります。これがアプリのパイプラインスタックサイズ計算にどのように寄与するか、詳しくはパイプラインスタックを参照してください。これは、アプリが保守的なデフォルトサイズに依存するのではなく、SetPipelineStackSize() を介してスタックサイズを設定したい場合にのみ呼び出される必要があります。

これは Ray 世代シェーダ、Hit グループ、Miss シェーダ、Callable シェーダに対してのみ有効です。

Hit グループでは、スタック サイズはそれを構成する個々のシェーダに対してクエリされる必要があります。Intersection シェーダ、Any hit シェーダ、Closest hit シェーダはそれぞれ異なるスタックサイズを要求される可能性があるためです。これらのシェーダがコンパイルされる方法は、それらを含む全体的なヒットグループによって影響を受ける可能性があるため、スタックサイズを直接照会することはできません。後述の pExportName パラメータには、ヒットグループ内の個々のシェーダを識別するための構文が含まれています。

この API はコレクションステートオブジェクトまたはレイトレーシング パイプラインステート オブジェクトのどちらかで呼び出すことができます。

ParameterDefinition`LPCWSTR pExportName`Shader entrypoint in the state object for which to retrieve stack size. For hit groups, an individual shader within the hit group must be specified as follows: "hitGroupNameshaderType", where hitGroupName is the entrypoint name for the hit group and shaderType is one of: intersection, closesthit or anyhit (all case sensitive). E.g. "myTreeLeafHitGroupanyhit"`Return value: UINT64`<p>Amount of stack in bytes required to invoke the shader. If the shader isn't fully resolved in the state object, or the shader is unknown or of a type for which a stack size isn't relevant (such as a [hit group](#hit-groups)) the return value is 0xffffffff. The reason for returning 32-bit 0xffffffff for a UINT64 return value is to ensure that bad return values don't get lost when summed up with other values as part of calculating an overall [Pipeline stack](#pipeline-stack) size.</p><p>This is only UINT64 at the API. In the DDI this is a UINT; drivers cannot return massive values to the runtime.</p>---

### GetPipelineStackSize

```C++
UINT64 GetPipelineStackSize();
```

現在のパイプライン スタック サイズを取得します。サイズの意味については、Pipeline stack を参照してください。

このコールと SetPipelineStackSize() はリエントラントではありません。つまり、別々のスレッドからどちらかまたは両方を呼び出す場合、アプリはそれ自体で同期する必要があります。

ParameterDefinition`Return value: UINT64`Current pipeline stack size in bytes. If called on non-executable state objects (e.g. collections), the return value is 0.---

### SetPipelineStackSize

```C++
void SetPipelineStackSize(UINT64 PipelineStackSizeInBytes);
```

現在のパイプラインスタックサイズを設定します。サイズの意味、デフォルト、サイズの選び方については、パイプラインスタックを参照してください。このメソッドは、オプションでレイトレーシングのパイプラインの状態について呼び出すことができます。

この呼び出しと GetPipelineStackSize() または DispatchRays() を介したレイトレーシング パイプライン ステート オブジェクトの使用は、リエントラントではありません。これは、別々のスレッドからこれらのいずれかを呼び出す場合、アプリはそれ自体で同期する必要があることを意味します。コマンドリストに記録された時点で、任意の DispatchRays() 呼び出しまたは ExecuteIndirect() 呼び出しは、最新のスタックサイズ設定を使用します。同様に、GetPipelineStackSize()コールは、常に最新のスタックサイズ設定を返します。

ランタイムは、レイトレーシング パイプライン以外のステート オブジェクト（コレクションなど）に対する呼び出しをドロップします。

ParameterDefinition`UINT64 PipelineStackSizeInBytes`<p>Stack size in bytes to use during pipeline execution for each shader thread (of which there can be many thousands in flight on the GPU).</p><p>If the value is \>= 0xffffffff (max 32-bit UINT) the runtime drops the call (debug layer will print an error) as this is likely the result of summing up invalid stack sizes returned from [GetShaderStackSize()](#getshaderstacksize) called with invalid parameters (which return 0xffffffff). In this case the previously set stack size (or default) remains.</p><p>This is only UINT64 at the API. In the DDI this is a UINT; as described above, massive stack sizes will not be requested from drivers.</p>---

## Additional resource states

```C++
D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE = 0x400000
```

この状態については、アクセラレーション構造更新制約の説明を参照してください。

---

## Additional root signature flags

---

### D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE

```C++
typedef enum D3D12_ROOT_SIGNATURE_FLAGS
{
    D3D12_ROOT_SIGNATURE_FLAG_NONE = 0x0,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT = 0x1,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_VERTEX_SHADER_ROOT_ACCESS = 0x2,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS = 0x4,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS = 0x8,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS = 0x10,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_PIXEL_SHADER_ROOT_ACCESS = 0x20,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_STREAM_OUTPUT = 0x40,
    D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE = 0x80,
} D3D12_ROOT_SIGNATURE_FLAGS;
```

`D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE` indicates the root
signature is to be used with raytracing shaders to define resource
bindings sourced from shader records in [shader tables](#shader-tables).
This flag cannot be combined with other root signature flags (list shown
above) that are all related to the graphics pipeline -- they don't make
sense together. The absence of the flag means the root signature can be
used with graphics or compute, where the compute version is also shared
with raytracing's (global) root signature.

ローカルルートシグネチャは、ルートシグネチャが行うルートパラメータの数に関する制限を持ちません。

ルートシグネチャの 2 つのクラスの間のこの区別は、各レイアウト の実装が異なる可能性があるため、ドライバにとって便利です（一方は CommandList からルート引数を調達し、他方はシェーダテーブルからそれらを調達 する）。

---

### Note on shader visibility

レイトレーシングで使用されるルート署名は、ローカルルートシグネチャ対ルート 署名で説明したように、コマンドリストの状態をコンピュートと共有します。そのため、適用されるルート引数シェーダの可視性は `D3D12_SHADER_VISIBILITY_ALL` のみであり、これは compute コマンドリストの状態の一部として設定されたルート引数がレイト レーシングにも可視であることを意味します。

ローカルルートシグネチャも `D3D12_SHADER_VISIBILITY_ALL` のみを使用することができます。

言い換えれば、ルート署名とローカルルート署名の両方について、シェー ダ可視性フラグで絞り込む面白いことは何もありません -- ローカルルート 引数は、単にすべてのレイトレーシングシェーダ（およびルート署名のための計算） に常に可視であるだけです。

---

## Additional SRV type

Acceleration structures are declared in HLSL via the
[RaytracingAccelerationStructure](#raytracingaccelerationstructure)
resource type, which can then be passed into [TraceRay()](#traceray) and [RayQuery::TraceRayInline()](#rayquery-tracerayinline).
From the API, these are bound either:

- この場合、他のルートディスクリプタ SRV と区別するために特別な表示は必要ありません。

- レイペイロードを読み込み、変更します。(inout payload_t rayPayload)

ディスクリプターヒープベースの加速構造 SRV を作成するとき、メモリロケーションは以下に示すビュー記述（ `D3D12_RAYTRACING_ACCELERATION_STRUCTURE_SRV` ）から GPUVA として来るので、リソースパラメータは NULL でなければなりません。例：CreateShaderResourceView(NULL,pViewDesc).

```C++
typedef struct D3D12_RAYTRACING_ACCELERATION_STRUCTURE_SRV
{
    D3D12_GPU_VIRTUAL_ADDRESS Location;
} D3D12_RAYTRACING_ACCELERATION_STRUCTURE_SRV;

typedef enum D3D12_SRV_DIMENSION {
    D3D12_SRV_DIMENSION_UNKNOWN = 0,
    D3D12_SRV_DIMENSION_BUFFER = 1,
    D3D12_SRV_DIMENSION_TEXTURE1D = 2,
    D3D12_SRV_DIMENSION_TEXTURE1DARRAY = 3,
    D3D12_SRV_DIMENSION_TEXTURE2D = 4,
    D3D12_SRV_DIMENSION_TEXTURE2DARRAY = 5,
    D3D12_SRV_DIMENSION_TEXTURE2DMS = 6,
    D3D12_SRV_DIMENSION_TEXTURE2DMSARRAY = 7,
    D3D12_SRV_DIMENSION_TEXTURE3D = 8,
    D3D12_SRV_DIMENSION_TEXTURECUBE = 9,
    D3D12_SRV_DIMENSION_TEXTURECUBEARRAY = 10,
    D3D12_SRV_DIMENSION_RAYTRACING_ACCELERATION_STRUCTURE = 11,
} D3D12_SRV_DIMENSION;

typedef struct D3D12_SHADER_RESOURCE_VIEW_DESC
{
    DXGI_FORMAT Format;
    D3D12_SRV_DIMENSION ViewDimension;
    UINT Shader4ComponentMapping;
    union
    {
        D3D12_BUFFER_SRV Buffer;
        D3D12_TEX1D_SRV Texture1D;
        D3D12_TEX1D_ARRAY_SRV Texture1DArray;
        D3D12_TEX2D_SRV Texture2D;
        D3D12_TEX2D_ARRAY_SRV Texture2DArray;
        D3D12_TEX2DMS_SRV Texture2DMS;
        D3D12_TEX2DMS_ARRAY_SRV Texture2DMSArray;
        D3D12_TEX3D_SRV Texture3D;
        D3D12_TEXCUBE_SRV TextureCube;
        D3D12_TEXCUBE_ARRAY_SRV TextureCubeArray;
        D3D12_RAYTRACING_ACCELERATION_STRUCTURE_SRV RaytracingAccelerationStructure;
    };
} D3D12_SHADER_RESOURCE_VIEW_DESC;
```

---

## Constants

```C++
#define D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BYTE_ALIGNMENT 256

#define D3D12_RAYTRACING_INSTANCE_DESC_BYTE_ALIGNMENT 16

#define D3D12_RAYTRACING_TRANSFORM3X4_BYTE_ALIGNMENT 16

#define D3D12_RAYTRACING_MAX_ATTRIBUTE_SIZE_IN_BYTES 32

#define D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT 32

#define D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT 64

#define D3D12_RAYTRACING_AABB_BYTE_ALIGNMENT 8

#define D3D12_RAYTRACING_MAX_DECLARABLE_TRACE_RECURSION_DEPTH 31

#define D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES 32
```

---

# HLSL

---

## Types, enums, subobjects and concepts

---

### Ray flags

Ray flags are passed to [TraceRay()](#traceray) or [RayQuery::TraceRayInline()](#rayquery-tracerayinline) to override
transparency, culling, and early-out behavior. For an example, see
[here](#flags-per-ray).

Ray flags are also a template paramter to [RayQuery](#rayquery) objects to enable static specialization - runtime behavior of [RayQuery::TraceRayInline()](#rayquery-tracerayinline) takes on the OR of dynamic and static/template ray falgs.

Shaders that interact with rays can query the
current flags via [RayFlags()](#rayflags) or [RayQuery::RayFlags()](#rayquery-rayflags) intrinsics.

```C++
enum RAY_FLAG : uint
{
    RAY_FLAG_NONE = 0x00,
    RAY_FLAG_FORCE_OPAQUE = 0x01,
    RAY_FLAG_FORCE_NON_OPAQUE = 0x02,
    RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH = 0x04,
    RAY_FLAG_SKIP_CLOSEST_HIT_SHADER = 0x08,
    RAY_FLAG_CULL_BACK_FACING_TRIANGLES = 0x10,
    RAY_FLAG_CULL_FRONT_FACING_TRIANGLES = 0x20,
    RAY_FLAG_CULL_OPAQUE = 0x40,
    RAY_FLAG_CULL_NON_OPAQUE = 0x80,
    RAY_FLAG_SKIP_TRIANGLES = 0x100,
    RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES = 0x200,
};
```

(d3d12.h には便宜上、同等の `D3D12_RAY_FLAG_*` が定義されています)

ValueDefinition`RAY_FLAG_NONE`No options selected.`RAY_FLAG_FORCE_OPAQUE`<p>All ray-primitive intersections encountered in a raytrace are treated as opaque. So no [any hit](#any-hit-shaders) shaders will be executed regardless of whether or not the hit geometry specifies [D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE](#d3d12_raytracing_geometry_flags), and regardless of the [instance flags](#d3d12_raytracing_instance_flags) on the instance that was hit.</p><p>In the case of [RayQuery::TraceRayInline()](#rayquery-tracerayinline), the impact of opacity isn't on shader invocations but instead on [TraceRayInline control flow](#tracerayinline-control-flow).</p> <p>Mutually exclusive to `RAY_FLAG_FORCE_NON_OPAQUE`, `RAY_FLAG_CULL_OPAQUE` and `RAY_FLAG_CULL_NON_OPAQUE`.</p>`RAY_FLAG_FORCE_NON_OPAQUE`<p>All ray-primitive intersections encountered in a raytrace are treated as non-opaque. So [any hit](#any-hit-shaders) shaders, if present, will be executed regardless of whether or not the hit geometry specifies [D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE](#d3d12_raytracing_geometry_flags), and regardless of the [instance flags](#d3d12_raytracing_instance_flags) on the instance that was hit.</p><p>In the case of [RayQuery::TraceRayInline()](#rayquery-tracerayinline), the impact of opacity isn't on shader invocations but instead on [TraceRayInline control flow](#tracerayinline-control-flow).</p> <p>Mutually exclusive to `RAY_FLAG_FORCE_OPAQUE`, `RAY_FLAG_CULL_OPAQUE` and `RAY_FLAG_CULL_NON_OPAQUE`.</p>`RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH`<p>The first ray-primitive intersection encountered in a raytrace automatically causes [AcceptHitAndEndSearch()](#accepthitandendsearch) to be called immediately after the any hit shader (including if there is no any hit shader).</p><p>The only exception is when the preceding [any hit](#any-hit-shaders) shader calls [IgnoreHit()](#ignorehit), in which case the ray continues unaffected (such that the next hit becomes another candidate to be the first hit. For this exception to apply, the any hit shader has to actually be executed. So if the any hit shader is skipped because the hit is treated as opaque (e.g. due to `RAY_FLAG_FORCE_OPAQUE` or [D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE](#d3d12_raytracing_geometry_flags) or [D3D12_RAYTRACING_INSTANCE_FLAG_OPAQUE](#d3d12_raytracing_instance_flags) being set), then [AcceptHitAndEndSearch()](#accepthitandendsearch) is called.</p><p>If a [closest hit](#closest-hit-shaders) shader is present at the first hit, it gets invoked (unless `RAY_FLAG_SKIP_CLOSEST_HIT_SHADER` is also present). The one hit that was found is considered "closest", even though other potential hits that might be closer on the ray may not have been visited.</p><p>A typical use for this flag is for shadows, where only a single hit needs to be found.</p><p>In the case of [RayQuery::TraceRayInline()](#rayquery-tracerayinline), the impact of this flag is similar to the above, except applied to [TraceRayInline control flow](#tracerayinline-control-flow), where after the first committed hit, either from fixed function intersection or from the shader, traversal completes.</p>`RAY_FLAG_SKIP_CLOSEST_HIT_SHADER`<p>Even if at least one hit has been committed, and the hit group for the closest hit contains a closest hit shader, skip execution of that shader.</p><p>One example where this flag could help is illustrated [here](#flags-per-ray).</p><p>This flag is not allowed (makes no sense for) [RayQuery::TraceRayInline()](#rayquery-tracerayinline) or [RayQuery](#rayquery)'s template parameter.</p>`RAY_FLAG_CULL_BACK_FACING_TRIANGLES`<p>Enables culling of back facing triangles. See [D3D12_RAYTRACING_INSTANCE_FLAGS](#d3d12_raytracing_instance_flags) for selecting which triangles are back facing, per-instance.</p><p>On instances that specify [D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_CULL_DISABLE](#d3d12_raytracing_instance_flags), this flag has no effect.</p><p>On geometry types other than [D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES](#d3d12_raytracing_geometry_type), this flag has no effect.</p><p>This flag is mutually exclusive to `RAY_FLAG_CULL_FRONT_FACING_TRIANGLES`.</p>`RAY_FLAG_CULL_FRONT_FACING_TRIANGLES`<p>Enables culling of front facing triangles. See [D3D12_RAYTRACING_INSTANCE_FLAGS](#d3d12_raytracing_instance_flags) for selecting which triangles are back facing, per-instance.</p><p>On instances that specify [D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_CULL_DISABLE](#d3d12_raytracing_instance_flags), this flag has no effect.</p><p> On geometry types other than [D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES](#d3d12_raytracing_geometry_type), this flag has no effect.</p><p>This flag is mutually exclusive to `RAY_FLAG_CULL_BACK_FACING_TRIANGLES`.</p>`RAY_FLAG_CULL_OPAQUE`<p>Culls all primitives that are considered opaque based on their [geometry](#d3d12_raytracing_geometry_flags) and [instance](#d3d12_raytracing_instance_flags) flags.</p><p>This flag is mutually exclusive to `RAY_FLAG_FORCE_OPAQUE`, `RAY_FLAG_FORCE_NON_OPAQUE`, and `RAY_FLAG_CULL_NON_OPAQUE`.`RAY_FLAG_CULL_NON_OPAQUE`<p>Culls all primitives that are considered non-opaque based on their [geometry](#d3d12_raytracing_geometry_flags) and [instance](#d3d12_raytracing_instance_flags) flags.</p><p>This flag is mutually exclusive to `RAY_FLAG_FORCE_OPAQUE`, `RAY_FLAG_FORCE_NON_OPAQUE`, and `RAY_FLAG_CULL_OPAQUE`.</p>`RAY_FLAG_SKIP_TRIANGLES`<p>Culls all triangles.</p><p>This flag is mutually exclusive to `RAY_FLAG_CULL_FRONT_FACING_TRIANGLES`, `RAY_FLAG_CULL_BACK_FACING_TRIANGLES` and `RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES`.</p><p>If instances specify [D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_CULL_DISABLE](#d3d12_raytracing_instance_flags) it has no effect, as that flag is meant for disabling front or back culling only. In other words `RAY_FLAG_SKIP_TRIANGLES` cannot be overruled.</p><p>This flag is only supported as of [Tier 1.1](#d3d12_raytracing_tier) implementations.</p>`RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES`<p>Culls all procedural primitives.</p><p>This flag is mutually exclusive to `RAY_FLAG_SKIP_TRIANGLES`.</p><p>This flag is only supported as of [Tier 1.1](#d3d12_raytracing_tier) implementations.</p>---

### Ray description structure

The RayDesc structure is passed to [TraceRay()](#traceray) or [RayQuery::TraceRayInline()](#rayquery-tracerayinline) to define the
origin, direction, and extents of the ray.

```C++
struct RayDesc
{
    float3 Origin;
    float TMin;
    float3 Direction;
    float TMax;
};
```

---

### Raytracing pipeline flags

レイトレーシング・パイプラインの config1 サブオブジェクトで使用されるフラグです。

```C++
enum RAYTRACING_PIPELINE_FLAG : uint
{
    RAYTRACING_PIPELINE_FLAG_NONE                         = 0x0,
    RAYTRACING_PIPELINE_FLAG_SKIP_TRIANGLES               = 0x100,
    RAYTRACING_PIPELINE_FLAG_SKIP_PROCEDURAL_PRIMITIVES   = 0x200,
};

```

フラグの定義については、D3D12 API の相当する D3D12_RAYTRACING_PIPELINE_FLAGS を参照してください。

---

### RaytracingAccelerationStructure

RaytracingAccelerationStructure は、HLSL で宣言できるリソースタイプです。これは、記述子テーブルまたはルート記述子 SRV の中の生バッファ SRV として束縛されます。HLSL での宣言は以下の通りです。

```C++
RaytracingAccelerationStructure MyScene[] : register(t3,space1);
```

この例では、acceleration structure の無制限サイズの配列を示していますが、これは、ルートディスクリプタがインデックス化できないので、ディスクリプタヒープから来ることを意味しています。

The RaytracingAccelerationStructure (for instance MyScene[i]) is
passed to [TraceRay()](#traceray) or [RayQuery::TraceRayInline()](#rayquery-tracerayinline) to indicate the top-level acceleration
resource built using [BuildRaytracingAccelerationStructure()](#buildraytracingaccelerationstructure).
It is an opaque resource with no methods available to shaders.

---

### Subobject definitions

サブオブジェクトは、API を使用して実行時に作成するだけでなく、HLSL で定義し、コンパイルされた DXIL ライブラリで利用できるようにすることができます。

---

#### Hit group

ヒットグループは、単一のシェーダエントリー関数ではなく、名前（文字列）で参照される 0 または 1 の intersection、anyhit、closehit のシェーダのグループです。シェーダタイプを省略する場合は、空の文字列を使用します。

例

```C++
HitGroup my_group_name("intersection_main", "anyhit_main",
"closesthit_main");
```

---

#### Root signature

レイトレーシングのパイプラインでグローバルに使用できる、または名前によってシェーダに関連付けられる、名前付きルート署名。ルート署名は DispatchRays 呼び出し内のすべてのシェーダに対してグローバルです。

```C++
RootSignature my_rs_name("root signature definition");
```

---

#### Local root signature

シェーダーと関連付けることができる名前付きローカルルートシグネチャ。ローカルルートシグネチャは、シェーダーテーブルのシェーダーレコードから読み取られる追加のルート引数の構造を定義します。

```C++
LocalRootSignature my_local_rs_name("local root signature definition");
```

---

#### Subobject to entrypoint association

ローカルルートシグネチャのような 1 つのサブオブジェクトとシェーダーエントリーポイントのリストとの間の関連付け。サブオブジェクトは文字列の名前で参照され、エントリポイントのリストは文字列の関数名のセミコロンで区切られたリストとして提供される。

```C++
SubobjectToEntrypointAssociation
my_association_name("subobject_name","function1;function2;function3");
```

---

#### Raytracing shader config

レイペイロードと交差点アトリビュートの最大サイズをバイトで定義します。API の同等品を参照してください。d3d12_raytracing_shader_config を参照してください。

```C++
RaytracingShaderConfig shader_config_name(maxPayloadSizeInBytes,maxAttributeSizeInBytes);
```

---

#### Raytracing pipeline config

TraceRay() recursion depth の最大値を定義します。同等の API を参照。D3D12_RAYTRACE_PIPLINE_CONFIG.

```C++
RaytracingPipelineConfig config_name(maxTraceRecursionDepth);
```

---

#### Raytracing pipeline config1

レイトレーシングのパイプラインフラグと同様に、最大の TraceRay() recursion depth を定義します。同等の API を参照してください。d3d12_raytracing_pipeline_config1 を参照してください。

```C++
RaytracingPipelineConfig1 config_name(maxTraceRecursionDepth,
                                      RAYTRACING_PIPELINE_CONFIG_FLAG_*);
```

利用可能なフラグ( `RAYTRACING_PIPELINE_CONFIG_FLAG_*` )は、以下で定義されています。レイトレーシング・パイプライン・フラグ(Raytracing pipeline flags)。

このサブオブジェクトは、Tier 1.1 のレイトレーシングをサポートするデバイスでのみ利用可能です。

---

### Intersection attributes structure

Intersection 属性は、2 つのソースのうちの 1 つから得られます。

**(1)** Triangle geometry via fixed-function triangle intersection. In this
case the structure used is the following:

```C++
struct BuiltInTriangleIntersectionAttributes
{
    float2 barycentrics;
};
```

[any hit](#any-hit-shaders) and [closest hit](#closest-hit-shaders) shaders invoked using fixed-function triangle
intersection must use this structure for hit attributes.

Given attributes `a0`, `a1` and `a2` for the 3 vertices of a triangle,
`barycentrics.x` is the weight for `a1` and `barycentrics.y` is the weight
for `a2`. For example, the app can interpolate by doing: `a = a0 + barycentrics.x \* (a1-a0) + barycentrics.y\* (a2 -- a0)`.

**(2)** Intersection with axis-aligned bounding boxes for procedural
primitives in the raytracing acceleration structure triggers an
intersection shader. That shader provides a user-defined
intersection attribute structure to the [ReportHit()](#reporthit)
call. The any hit and closest hit shaders bound in the same hit
group with this intersection shader must use the same structure for
hit attributes, even if the attributes are not referenced. The
maximum attribute structure size is 32 bytes
([D3D12_RAYTRACING_MAX_ATTRIBUTE_SIZE_IN_BYTES](#constants)).

---

### Ray payload structure

これは、TraceRay() 呼び出しの inout 引数として、およびレイペイロードにアクセスできるシェーダタイプ (any hit, closest hit, and miss shaders) の inout パラメータとして提供されるユーザ定義の構造体です。レイペイロードにアクセスするすべてのシェーダは、元の TraceRay() 呼び出しで提供されたものと同じ構造体を使用する必要があります。これらのシェーダがレイ ペイロードを全く参照しない場合でも、元の TraceRay() 呼び出しと同じペイロードを指定する必要があります。

シェーダモデル 6.6 および 6.7 で導入されたペイロードメンバに対するアノテーションの議論については、ペイロードアクセス修飾子を参照してください。

---

### Call parameter structure

これは、CallShader() 呼び出しの inout 引数として、また Callable シェーダの inout パラメータとして提供されるユーザ定義の構造体です。callable シェーダで使用される構造体タイプは、対応する CallShader() 呼び出しに提供される構造体と一致する必要があります。

---

## Shaders

これらのシェーダはライブラリ（ターゲット lib_6_3）にコンパイルされた関数で、シェーダ関数上の属性 [shader("shadertype")] で識別されます。

各シェーダタイプで何が許可されているかは、イントリンシックとシステム値イントリンシックを参照してください。

グラフィックスまたはコンピュートシェーダタイプでサポートされている特定の機能は、レイトレーシングのシェーダタイプではサポートされていません。独立性に起因するシェーダの制限を参照してください。

---

### Ray generation shader

シェーダタイプを参照してください。 `raygeneration`

概要はこちら

Ray generation shaders call [TraceRay()](#traceray) to generate rays (aside: [RayQuery::TraceRayInline()](#rayquery-tracerayinline) works too, as it can be called from anywhere).
The initial user-defined ray payload for each ray is provided to the
[TraceRay()](#traceray) call site. [CallShader()](#callshader) may also
be used in ray generation shaders to invoke [callable shaders](#callable-shaders).

大まかな例です。

```C++
struct SceneConstantStructure { ... };

ConstantBuffer<SceneConstantStructure> SceneConstants;
RaytracingAccelerationStructure MyAccelerationStructure : register(t3);
struct MyPayload { ... };

[shader("raygeneration")]
void raygen_main()
{
    RayDesc myRay = {
        SceneConstants.CameraOrigin,
        SceneConstants.TMin,
        computeRayDirection(SceneConstants.LensParams, DispatchRaysIndex(),
        DispatchRaysDimensions()),
        SceneConstants.TMax
        };

    MyPayload payload = { ... }; // init payload

    TraceRay(
        MyAccelerationStructure,
        SceneConstants.RayFlags,
        SceneConstants.InstanceInclusionMask,
        SceneConstants.RayContributionToHitGroupIndex,
        SceneConstants.MultiplierForGeometryContributionToHitGroupIndex,
        SceneConstants.MissShaderIndex,
        myRay,
        payload);

    WriteFinalPixel(DispatchRaysIndex(), payload);
}
```

---

### Intersection shader

シェーダタイプ。 `intersection`

概要はこちら

カスタム交差点プリミティブを実装するために使用される交差点シェーダは、関連するバウンディングボリューム（バウンディングボックス）と交差するレイのために呼び出されます。交差点シェーダはレイペイロードにアクセスすることはできませんが、ReportHit()コールを通して各ヒットに対する交差点属性を定義します。レイフラグ `RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH` が設定されている場合、または AcceptHitAndEndSearch() が任意のヒットシェーダから呼び出された場合、 ReportHit() の処理は交差点シェーダを早期に停止することがあります。それ以外の場合は、ヒットが受け入れられた場合は真を、拒否された場合は偽を返します（詳細は ReportHit() を参照）。これは、もし Any Hit シェーダが存在すれば、制御が条件付きで交差点シェーダに戻る前に実行されなければならないことを意味します。

大まかな例です。

```C++
struct CustomPrimitiveDef { ... };
struct MyAttributes { ... };
struct CustomIntersectionIterator {...};

void InitCustomIntersectionIterator(CustomIntersectionIterator it) {...}

bool IntersectCustomPrimitiveFrontToBack(
    CustomPrimitiveDef prim,
    inout CustomIntersectionIterator it,
    float3 origin, float3 dir,
    float rayTMin, inout float curT,
    out MyAttributes attr);

[shader("intersection")]
void intersection_main()
{
    float THit = RayTCurrent();
    MyAttributes attr;
    CustomIntersectionIterator it;

    InitCustomIntersectionIterator(it);

    while(IntersectCustomPrimitiveFrontToBack(
          CustomPrimitiveDefinitions[LocalConstants.PrimitiveIndex],
          it, ObjectRayOrigin(), ObjectRayDirection(),
          RayTMin(), THit, attr))
    {
        // Exit on the first hit. Note that if the ray has
        // RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH or an
        // anyhit shader is used and calls AcceptHitAndEndSearch(),
        // that would also fully exit this intersection shader (making
        // the "break" below moot in that case).
        if (ReportHit(THit, /*hitKind*/ 0, attr) &&
             (RayFlags() & RAY_FLAG_FORCE_OPAQUE))
            break;
    }
}
```

---

### Any hit shader

シェーダタイプ `anyhit`

概要はこちら

交差点が不透明でないとき、エニーヒットシェーダが呼び出されます。エニーヒットシェーダは、ペイロードパラメータと、それに続くアトリビュートパラメータを宣言する必要があります。それぞれ、TraceRay と ReportHit で使用されるタイプと一致するユーザ定義の構造体タイプでなければなりません（または、固定関数の三角形交差が使用されている場合は BuiltInIntersectionAttributes 構造体です）。

任意のヒットシェーダは、以下のようなことを行うことができます。

- 交差点属性の読み込み: (in attr_t attributes)

- AcceptHitAndEndSearch() を呼び出し、現在のヒットを受け入れ、任意のヒットシェーダを終了し、交差点シェーダを終了し（あれば）、これまでで最も近いヒットに最も近いヒットシェーダを実行します（アクティブな場合）。

- IgnoreHit()を呼び出し、任意のヒットシェーダを終了し、ReportHit()呼び出し部位から false を返す交差点シェーダ（現在実行中の場合）に制御を戻すことを含め、ヒットの検索を継続するようにシステムに指示します。

- 現在のヒットを受け入れ、交差点シェーダに制御を戻すことを含め、ヒットの検索を継続するようシステムに指示し、ヒットが受け入れられたことを示すために ReportHit()呼び出しサイトで true を返します。

- レイペイロードを読み込んで変更します。(inout payload_t rayPayload)

IgnoreHit() または AcceptHitAndEndSearch() によって Any Hit シェーダの呼び出しが終了しても、これまでにレイペイロードに加えられたすべての修正は保持されなければなりません。

大まかな例です。

```C++
[shader("anyhit")]
void anyhit_main( inout MyPayload payload, in MyAttributes attr )
{
  float3 hitLocation = ObjectRayOrigin() + ObjectRayDirection() *
      RayTCurrent();

  float alpha = computeAlpha(hitLocation, attr, ...);

  // Processing shadow and only care if a hit is registered?
  if (TerminateShadowRay(alpha))
      AcceptHitAndEndSearch(); // aborts function

  // Save alpha contribution and ignoring hit?
  if (SaveAndIgnore(payload, RayTCurrent(), alpha, attr, ...))
      IgnoreHit(); // aborts function

  // do something else
  // return to accept and update closest hit
}
```

---

### Closest hit shader

シェーダタイプ。 `closesthit`

概要はこちら

最も近いヒットが決定されたとき、またはレイ交差探索が終了したとき、最も近いヒットシェーダが呼び出されます(有効な場合)。ここで、サーフェイスシェーディングと追加のレイ生成が一般的に行われ ます。最接近ヒットシェーダは、ペイロードパラメータと、それに続くアトリビュートパラメータを宣言する必要があります。それぞれ、TraceRay と ReportHit で使用されるタイプと一致するユーザ定義の構造体でなければなりません（または、固定関数の三角形交差が使用される場合は BuiltInIntersectionAttributes 構造体）。

最接近シェーダは可能です。

- 交差点属性の読み込み: (in attr_t attributes)

- CallShader() と TraceRay() を使って、さらに作業をスケジュールし、結果を読み返します。

- TraceRay 呼び出しの最初に、 `write(caller)` とマークされたペイロードタイプのフィールドは、TraceRay に渡されたペイロード引数から実際のペイロードにコピーされます。実際のペイロードの他の全てのフィールドは未定義の内容である。

大まかな例です。

```C++
[shader("closesthit")]
void closesthit_main(inout MyPayload payload, in MyAttributes attr)
{
    CallShader( ... ); // maybe

    // update payload for surface
    // trace reflection
    float3 worldRayOrigin = WorldRayOrigin() + WorldRayDirection() *
        RayTCurrent();

    float3 worldNormal = mul(attr.normal, (float3x3)ObjectToWorld3x4());
    RayDesc reflectedRay = { worldRayOrigin, SceneConstants.Epsilon,
                              ReflectRay(WorldRayDirection(), worldNormal),
                              SceneConstants.TMax };

    TraceRay(MyAccelerationStructure,
            SceneConstants.RayFlags,
            SceneConstants.InstanceInclusionMask,
            SceneConstants.RayContributionToHitGroupIndex,
            SceneConstants.MultiplierForGeometryContributionToHitGroupIndex,
            SceneConstants.MissShaderIndex,
            reflectedRay,
            payload);

    // Combine final contributions into ray payload
    // this ray query is now complete.

    // Alternately, could look at data in payload result based on that make
    // other TraceRay calls. No constraints on the code structure.
}
```

上記の "rayOrigin + rayDirection \* RayTCurrent()" によって現在のレイのヒット位置を計算する例の代わりに、バリセントリックと頂点位置を使用してサーフェスパラメトリゼーションによってヒット位置を計算することが可能です。前者は計算が速いですが、浮動小数点エラーが発生しやすく、エラーが発生するとレイ方向に沿って位置がオフセットされ、サーフェスから遠ざかることがよくあります。これは特に、大きな RayTCurrent() 値の場合に当てはまります。もう一つの方法であるサーフェスパラメトリゼーションは、計算誤差があっても計算された位置はほとんどサーフェスに沿って移動するのでより正確ですが、頂点の位置をロードしてより多くの計算を行う必要があるのでより高価です。精度が重要な場合は、余分なオーバーヘッドに価値があるかもしれません。

---

### Miss shader

シェーダタイプ。 `miss`

概要はこちら。ミスシェーダは、TraceRay() に供給されたものと一致するユーザ定義の構造体型のペイロードパラメータを含まなければなりません。

交差が見つからなかったり、受け入れられたりすると、ミス・シェーダが起動されます。これは、背景や空のシェーディングに便利です。ミスシェーダは CallShader()と TraceRay()を使用して、より多くの作業をスケジューリングすることができます。

大まかな例です。

```C++
[shader("miss")]
void miss_main(inout MyPayload payload)
{
  // Use ray system values to compute contributions of background,
  // sky,etc...

  // Combine contributions into ray payload
  CallShader( ... ); // maybe

  TraceRay( ... ); // maybe

  // this ray query is now complete
}
```

---

### Callable shader

シェーダタイプ。 `callable`

概要はこちら

呼び出し可能なシェーダは、CallShader() 組み込み関数を使って他のシェーダから呼び出されます。CallShader()コールサイトで提供されるパラメータ構造は、 DispatchRays() API を通して提供される呼び出し可能シェーダテーブルへの 要求インデックスによって指される呼び出し可能シェーダで使われるパ ラメータ構造と一致しなければなりません。呼び出し可能なシェーダはこのパラメータを inout として宣言する必要があります。さらに、呼び出し可能なシェーダは起動インデックスと次元入力を読み取ることができます。

大まかな例です。

```C++
[shader("callable")]
void callable_main(inout MyParams params)
{
    // Perform some common operations and update params
    CallShader( ... ); // maybe
}
```

---

## Intrinsics

**intrinsics \\ shaders**ray generationintersectionany hitclosest hitmisscallableall shaders including graphics/compute[CallShader()](#callshader)\*\*\*\*[TraceRay()](#traceray)\*\*\*[ReportHit()](#reporthit)\*[IgnoreHit()](#ignorehit)\*[AcceptHitAndEndSearch()](#accepthitandendsearch)\*---

### CallShader

この組込み関数定義は、次の関数テンプレートと同等です。

```C++
template<param_t>
void CallShader(uint ShaderIndex, inout param_t Parameter);
```

ParameterDefinition`uint ShaderIndex`Provides index into the callable shader table supplied through the [DispatchRays()](#dispatchrays) API. See [Callable shader table indexing](#callable-shader-table-indexing).`inout param_t Parameter`The user-defined parameters to pass to the callable shader. This parameter structure must match the parameter structure used in the callable shader pointed to in the shader table.---

### TraceRay

acceleration structure でヒットするようにレイを送る。該当する場合、様々なタイプのシェーダ呼び出しを含む。TraceRay() のコントロールフローを参照してください。

この組込み関数定義は、次の関数テンプレートと同等です。

```C++
Template<payload_t>
void TraceRay(RaytracingAccelerationStructure AccelerationStructure,
    uint RayFlags,
    uint InstanceInclusionMask,
    uint RayContributionToHitGroupIndex,
    uint MultiplierForGeometryContributionToHitGroupIndex,
    uint MissShaderIndex,
    RayDesc Ray,
    inout payload_t Payload);
```

ParameterDefinition`RaytracingAccelerationStructure AccelerationStructure`Top-level acceleration structure to use. Specifying a NULL acceleration structure forces a miss.`uint RayFlags`Valid combination of [Ray flags](#ray-flags). Only defined ray flags are propagated by the system, e.g. visible to the [RayFlags()](#rayflags) shader intrinsic.`uint InstanceInclusionMask`<p>Bottom 8 bits of InstanceInclusionMask are used to include/reject geometry instances based on the InstanceMask in each [instance](#d3d12_raytracing_instance_desc):</p><p>`if(!((InstanceInclusionMask & InstanceMask) & 0xff)) { ignore intersection }`</p>`uint RayContributionToHitGroupIndex`Offset to add into [Addressing calculations within shader tables](#addressing-calculations-within-shader-tables) for hit group indexing. Only the bottom 4 bits of this value are used.`uint MultiplierForGeometryContributionToShaderIndex`Stride to multiply by GeometryContributionToHitGroupIndex (which is just the 0 based index the geometry was supplied by the app into its bottom-level acceleration structure). See [Addressing calculations within shader tables](#addressing-calculations-within-shader-tables) for hit group indexing. Only the bottom 4 bits of this multiplier value are used.`uint MissShaderIndex`Miss shader index in [Addressing calculations within shader tables](#addressing-calculations-within-shader-tables). Only the bottom 16 bits of this value are used.`RayDesc Ray`[Ray](#ray-description-structure) to be traced. See [Ray-extents](#ray-extents) for bounds on valid ray parameters.`inout payload_t Payload`User defined ray payload accessed both for both input and output by shaders invoked during raytracing. After TraceRay completes, the caller can access the payload as well.---

### ReportHit

この組込み関数の定義は次の関数テンプレートと同等です。

```C++
template<attr_t>
bool ReportHit(float THit, uint HitKind, attr_t Attributes);
```

ParameterDefinition`float THit`The parametric distance of the intersection.`uint HitKind`A value used to identify the type of hit. This is a user-specified value in the range of 0-127. The value can be read by [any hit](#any-hit-shaders) or [closest hit](#closest-hit-shaders) shaders with the [HitKind()](#hitkind) intrinsic.`attr_t Attributes`Intersection attributes. The type attr_t is the user-defined intersection attribute structure. See [Intersection attributes structure](#intersection-attributes-structure).ReportHit は、ヒットが受け入れられた場合に true を返します。THit が現在のレイ間隔の外にある場合、または任意のヒットシェーダーが IgnoreHit() を呼び出した場合、ヒットは拒否されます。現在のレイ間隔は RayTMin() と RayTCurrent() によって定義されます。

---

### IgnoreHit

```C++
void IgnoreHit();
```

ヒットを拒否してシェーダを終了させるために Any Hit シェーダで使用されます。ヒット検索は、現在のヒットの距離 (hitT) とアトリビュートをコミットせずに続行されます。交差点シェーダ内の ReportHit()コール(もしあれば)は false を返します。any hit シェーダでこの時点までにレイペイロードに加えられた変更はすべて保存されます。

---

### AcceptHitAndEndSearch

```C++
void AcceptHitAndEndSearch();
```

Any hit シェーダで、現在のヒット (hitT とアトリビュート) をコミットし、レイのさらなるヒットの検索を停止するために使用されます。交差点シェーダが実行されている場合、それは停止します。実行は、これまでに記録された最も近いヒットを持つ最も近いヒットシェーダ（有効な場合）に渡されます。

---

## System value intrinsics

システム値は、シェーダー関数シグネチャに特別なセマンティクスを持つパラメー タを含めるのではなく、特別な組込み関数を使用することで取得されます。

次の表は、システム値の組込み関数がどこにあるかを示しています。

**values \\ shaders**ray generationintersectionany hitclosest hitmisscallable**\*Ray dispatch system values:**\_uint3 [DispatchRaysIndex()](#dispatchraysindex)\*\*\*\*\*\*uint3 [DispatchRaysDimensions()](#dispatchraysdimensions)\*\*\*\*\*\***\*Ray system values:**_float3 [WorldRayOrigin()](#worldrayorigin)\*\*\*\*float3 [WorldRayDirection()](#worldraydirection)\*\*\*\*float [RayTMin()](#raytmin)\*\*\*\*float [RayTCurrent()](#raytcurrent)\*\*\*\*uint [RayFlags()](#rayflags)\*\*\*\*_**Primitive/object space system values:**_uint [InstanceIndex()](#instanceindex)\*\*\*uint [InstanceID()](#instanceid)\*\*\*uint [GeometryIndex()](#geometryindex)<br>(requires [Tier 1.1](#d3d12_raytracing_tier) implementation)\*\*\*uint [PrimitiveIndex()](#primitiveindex)\*\*\*float3 [ObjectRayOrigin()](#objectrayorigin)\*\*\*float3 [ObjectRayDirection()](#objectraydirection)\*\*\*float3x4 [ObjectToWorld3x4()](#objecttoworld3x4)\*\*\*float4x3 [ObjectToWorld4x3()](#objecttoworld4x3)\*\*\*float3x4 [WorldToObject3x4()](#worldtoobject3x4)\*\*\*float4x3 [WorldToObject4x3()](#worldtoobject4x3)\*\*\*_**Hit specific system values:**\_uint [HitKind()](#hitkind)\*\*---

### Ray dispatch system values

起動システム値は、すべてのレイトレーシングのシェーダタイプで利用可能な入力です。これらは、現在のシェーダインスタンスにつながったレイ生成シェーダインスタンスでの値を返します。

---

#### DispatchRaysIndex

DispatchRaysDimensions() システム値組込み関数で利用可能になった Width と Height の中の現在の x と y の位置。

```C++
uint3 DispatchRaysIndex();
```

---

#### DispatchRaysDimensions

DispatchRays() 呼び出しの元になった `D3D12_DISPATCH_RAYS_DESC` 構造体からの Width, Height, Depth の値です。

```C++
uint3 DispatchRaysDimensions();
```

---

### Ray system values

これらのシステム値は、ヒットグループとミスシェーダ内のすべてのシェーダに利用可能です。

---

#### WorldRayOrigin

現在のレイのワールドスペースの原点。

```C++
float3 WorldRayOrigin();
```

---

#### WorldRayDirection

現在のレイのワールド空間方向。

```C++
float3 WorldRayDirection();
```

---

#### RayTMin

レイのパラメトリックな始点を表す float です。

```C++
float RayTMin();
```

RayTMin は、次の式に従ってレイの始点を定義します。Origin + (Direction \* RayTMin)。Origin と Direction はワールド空間でもオブジェクト空間でもよく、その結果、ワールドまたはオブジェクト空間の始点になります。

RayTMin は TraceRay()を呼び出すときに定義され、その呼び出しの間、一定です。

---

#### RayTCurrent

これはレイの現在のパラメトリック終了点を表す浮動小数点数です。

```C++
float RayTCurrent();
```

RayTCurrent は、次の式に従って、レイの現在の終点を定義します。Origin + (Direction \* RayTCurrent)です。Origin と Direction は、ワールド空間でもオブジェクト空間でもかまいませんので、結果的にワールドまたはオブジェクト空間の終点になります。

RayTCurrent は RayDesc::TMax からの TraceRay() コールによって初期化され、トレースクエリ中に交差が報告され（any hit）、受け入れられると更新されます。

交差点シェーダでは、これまでに発見された最も近い交差点までの距離を表します。これは ReportHit() の後に、ヒットが受け入れられた場合に提供される THit 値に更新されます。

任意のヒットシェーダでは、報告されている現在の交差点までの距離を表します。

closest hit シェーダでは、これは、受け入れられた最も近い交差点までの距離を表します。

ミスシェーダでは、TraceRay()呼び出しに渡された TMax に等しくなります。

---

#### RayFlags

これは現在のレイフラグ(のみ)を含む uint です。 D3D12_RAYTRACING_PIPELINE_CONFIG1 によって外部で追加された可能性のあるフラグは表示されません。

```C++
uint RayFlags();
```

これは、例えば、交差点シェーダにおいて、アプリケーションが現在のレイのカリングフラグを見て、カスタム交差点コードで対応するカリングを適用したい場合に便利です。

---

### Primitive/object space system values

これらのシステム値は、プリミティブが交差のために選択されると、利用可能になります。これらのシステム値により、レイによって交差されるもの、オブジェクト空間のレイの原点と方向、オブジェクト空間とワールド空間間のトランスフォームマトリックスを特定することができます。

---

#### InstanceIndex

トップレベル構造における現在のインスタンスの自動生成されたインデックスです。

```C++
uint InstanceIndex();
```

---

#### InstanceID

最上位構造体内の bottom-level acceleration structure インスタンスの、ユーザー提供の InstanceID。

```C++
uint InstanceID();
```

---

#### GeometryIndex

ボトムレベル・アクセラレーション構造内の現在のジオメトリの自動生成されたインデックス。

このメソッドは Tier 1.1 の実装でのみ利用可能です。

TraceRay() の MultiplierForGeometryContributionToHitGroupIndex パラメータを 0 に設定すると、シェーダがジオメトリを区別するために GeometryIndex() に依存したい場合に、シェーダ テーブル インデックスに寄与しないようにすることができる。

```C++
uint GeometryIndex();
```

---

#### PrimitiveIndex

最下位の acceleration structure インスタンス内のジオメトリ内のプリミティブの自動生成されたインデックス。

```C++
uint PrimitiveIndex();
```

`D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES` の場合、これはジオメトリオブジェクト内の三角形のインデックスになります。

`D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS` では、ジオメトリ オブジェクトを定義する AABB 配列へのインデックスになります。

---

#### ObjectRayOrigin

現在のレイのオブジェクト空間原点です。Object-space は、現在の bottom-level acceleration structure の空間を指します。

```C++
float3 ObjectRayOrigin();
```

---

#### ObjectRayDirection

現在のレイのオブジェクト空間方向。オブジェクト空間は、現在の bottom-level acceleration structure の空間を参照します。

```C++
float3 ObjectRayDirection();
```

---

#### ObjectToWorld3x4

オブジェクト空間からワールド空間への変換のためのマトリックス。オブジェクト空間は、現在のボトムレベル・アクセラレーション・ストラクチャーの空間を指します。

```C++
float3x4 ObjectToWorld3x4();
```

`ObjectToWorld4x3()` との唯一の違いは、行列が転置されることです -- どちらか都合のよいほうを使用してください。

---

#### ObjectToWorld4x3

オブジェクト空間からワールド空間への変換のためのマトリックス。オブジェクト空間は、現在のボトムレベル・アクセラレーション・ストラクチャーの空間を指します。

```C++
float4x3 ObjectToWorld4x3();
```

`ObjectToWorld3x4()` との唯一の違いは、行列が転置されることです。

---

#### WorldToObject3x4

ワールド空間からオブジェクト空間への変換を行うための行列。オブジェクト空間とは、現在の最下層 acceleration structure の空間を指します。

```C++
float3x4 WorldToObject3x4();
```

WorldToObject4x3() との唯一の違いは, 行列が転置されることである -- どちらか都合の良い方を使用する.

---

#### WorldToObject4x3

ワールド空間からオブジェクト空間への変換を行うための行列。オブジェクト空間とは、現在の最下層 acceleration structure の空間を指します。

```C++
float3x4 WorldToObject4x3();
```

WorldToObject3x4()との唯一の違いは、行列が転置されることです -- どちらか都合の良い方を使用してください。

---

### Hit specific system values

---

#### HitKind

ReportHit()で HitKind として渡された値を返します。交差が固定関数の三角形交差によって報告された場合、HitKind は `HIT_KIND_TRIANGLE_FRONT_FACE` (254) または `HIT_KIND_TRIANGLE_BACK_FACE` (255) のいずれかになります。

(d3d12.h には便宜上、同等の `D3D12_HIT_KIND_*` が定義されています)

```C++
uint HitKind();
```

---

## RayQuery

```C++
RayQuery<RAY_FLAGS>
```

`RayQuery` represents the state of an [inline raytracing](#inline-raytracing) call into an acceleration structure via member function [RayQuery::TraceRayInline()](#rayquery-tracerayinline). It is "inline" in the sense that this form of raytracing doesn't automatically invoke other shader invocations during traversal as [TraceRay()](#traceray) does. The shader observes `RayQuery` to be in one of a set of states, each of which exposes relevant [methods](#rayquery-intrinsics) to retrieve information about the current state or continue the query into the acceleration structure.

`RayQuery` takes a template parameter on instantiation: a set of [RAY_FLAGS](#ray-flags) that are inline/literal at compile time. These flags enable implementations to produce more efficient inline code generation by drivers customized to the selected flags. For example, declaring `RayFlags<RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES> myRayQuery` means ray traversals done via `myRayQuery` will not enumerate procedural primitives. Applications can also custimize their code to rely on the impact of the selected flags - in this example not having to write code to handle the procedural primitives showing up. [RayQuery::TraceRayInline()](#rayquery-tracerayinline) also has a [RAY_FLAGS](#ray-flags) field which allows dynamic ray flags configuration - the template flags are OR'd with the dynamic flags during a raytrace.

シュードコードの例はこちらです。

The size of `RayQuery` is implementation specific and opaque. Shaders can have any number of live `RayQuery` objects during shader execution, but at the cost of an unknown (and relatively high) amount of hardware register storage, which can limit how many shaders can run in parallel. This unknown size is in contrast to traditional shader variables (e.g. `float4 foo`), whose register cost is obvious by the data type of the variable (e.g. 16 bytes for a `float4`).

`RayQuery` 型の変数が他の変数（一致するテンプレート仕様で宣言されていなければなりません）に割り当てられると、（クローンではなく）元の変数への参照が渡されるため、両者は同じ共有ステートマシンで動作することになります。 `RayQuery` 型の変数がパラメータとして関数に渡される場合、それは参照渡しされます。 `RayQuery` 型の変数が他の変数で上書きされた場合（代入など）、上書きされたオブジェクトは消えます。

> A RayQuery::Clone() intrinsic was considered to enable forking an in progress ray traversal. But this appeared to be of no value, lacking a known interesting scenario.
>
> A proposed feature that was cut is the ability for full [TraceRay()](#traceray) to return a [RayQuery](#rayquery) object. This would have been a middle ground between inline raytracing and the dynamic-shader-based form - e.g. apps could use dynamic any-hit shaders but then choose to do final hit/miss processing in the calling shader, without the dynamic shaders having to bother stuffing ray query metadata like hit distance in the ray payload - redundant information given that the system knows it. It turned out that for some implementations this would actually be slower than the application manually stuffing only needed values into the ray payload, and would require extra compilation for paths that need the ray query versus those that do not.
>
> This feature could come back in a more refined form in the future. This could be allowing the user to declare entries in the ray payload as system generated values for all query data (e.g. SV_CurrentT). Some implementations could choose to compile shaders (which see the payload declaration) to automatically store these values in the ray payload as declared, whereas other implementations would be free to pull these system values as needed from elsewhere if they are readily available (avoiding payload bloat).

---

### RayQuery intrinsics

`RayQuery` supports the methods (\*) in the following tables depending on its current state. The intrinsics are listed across several tables for readability (with entries repeated as appicable). Calling unsupported methods is invalid and produces undefined results.

The first table lists intrinsics available after the two steps in `RayQuery` initialization: First, declaring a [RayQuery](#rayquery) object (with [RayFlags](#ray-flags) template parameter), followed by calling [RayQuery::TraceRayInline()](#rayquery-tracerayinline) to initialize trace parameters but not yet begin the traversal. [RayQuery::TraceRayInline()](#rayquery-tracerayinline) can be called over again any time to initialize a new trace, discarding the current one regardlessof what state it is in.

**Intrinsic** \ StateNew `RayQuery` object`TraceRayInline()` was called[TraceRayInline()](#rayquery-tracerayinline)\*\*`bool` [Proceed()](#rayquery-proceed)\*[Abort()](#rayquery-abort)\*`COMMITTED_STATUS` [CommittedStatus()](#rayquery-committedstatus)\*The following table lists intrinsics available when [RayQuery::Proceed()](#rayquery-proceed) returned `TRUE`, meaning a type of hit candidate that requires shader evaluation has been found. Methods named Committed\*() in this table may not actually be available depending on the current [CommittedStatus()](#rayquery-committedstatus) (i.e what type of hit has been commited yet if any?) - this is further clarified in another table further below.

**Intrinsic** \ [CandidateType()](#rayquery-candidatetype)` HIT_CANDIDATE_NON_OPAQUE_TRIANGLE``HIT_CANDIDATE_PROCEDURAL_PRIMITIVE `[TraceRayInline()](#rayquery-tracerayinline)\*\*`bool` [Proceed()](#rayquery-proceed)\*\*[Abort()](#rayquery-abort)\*\*`CANDIDATE_TYPE` [CandidateType()](#rayquery-candidatetype)\*\*bool [CandidateProceduralPrimitiveNonOpaque()](#rayquery-candidateproceduralprimitivenonopaque)\*[CommitNonOpaqueTriangleHit()](#rayquery-commitnonopaquetrianglehit)\*[CommitProceduralPrimitiveHit(float t)](#rayquery-commitproceduralprimitivehit)\*`COMMITTED_STATUS` [CommittedStatus())](#rayquery-committedstatus)\*\*_**Ray system values:**\_uint [RayFlags()](#rayquery-rayflags)\*\*float3 [WorldRayOrigin()](#rayquery-worldrayorigin)\*\*float3 [WorldRayDirection()](#rayquery-worldraydirection)\*\*float [RayTMin()](#rayquery-raytmin)\*\*float [CandidateTriangleRayT()](#rayquery-candidatetrianglerayt)\*float [CommittedRayT()](#rayquery-committedrayt)\*\*_**Primitive/object space system values:**_uint [CandidateInstanceIndex()](#rayquery-candidateinstanceindex)\*\*uint [CandidateInstanceID()](#rayquery-candidateinstanceid)\*\*uint [CandidateInstanceContributionToHitGroupIndex()](#rayquery-candidateinstancecontributiontohitgroupindex)\*\*uint [CandidateGeometryIndex()](#rayquery-candidategeometryindex)\*\*uint [CandidatePrimitiveIndex()](#rayquery-candidateprimitiveindex)\*\*float3 [CandidateObjectRayOrigin()](#rayquery-candidateobjectrayorigin)\*\*float3 [CandidateObjectRayDirection()](#rayquery-candidateobjectraydirection)\*\*float3x4 [CandidateObjectToWorld3x4()](#rayquery-candidateobjecttoworld3x4)\*\*float4x3 [CandidateObjectToWorld4x3()](#rayquery-candidateobjecttoworld4x3)\*\*float3x4 [CandidateWorldToObject3x4()](#rayquery-candidateworldtoobject3x4)\*\*float4x3 [CandidateWorldToObject4x3()](#rayquery-candidateworldtoobject4x3)\*\*uint [CommittedInstanceIndex()](#rayquery-committedinstanceindex)\*\*uint [CommittedInstanceID()](#rayquery-committedinstanceid)\*\*uint [CommittedInstanceContributionToHitGroupIndex()](#rayquery-committedinstancecontributiontohitgroupindex)\*\*uint [CommittedGeometryIndex()](#rayquery-committedgeometryindex)\*\*uint [CommittedPrimitiveIndex()](#rayquery-committedprimitiveindex)\*\*float3 [CommittedObjectRayOrigin()](#rayquery-committedobjectrayorigin)\*\*float3 [CommittedObjectRayDirection()](#rayquery-committedobjectraydirection)\*\*float3x4 [CommittedObjectToWorld3x4()](#rayquery-committedobjecttoworld3x4)\*\*float4x3 [CommittedObjectToWorld4x3()](#rayquery-committedobjecttoworld4x3)\*\*float3x4 [CommittedWorldToObject3x4()](#rayquery-committedworldtoobject3x4)\*\*float4x3 [CommittedWorldToObject4x3()](#rayquery-committedworldtoobject4x3)\*\*_**Hit specific system values:**\_float2 [CandidateTriangleBarycentrics()](#rayquery-candidatetrianglebarycentrics)\*bool [CandidateTriangleFrontFace()](#rayquery-candidatetrianglefrontface)\*float2 [CommittedTriangleBarycentrics()](#rayquery-committedtrianglebarycentrics)\*\*bool [CommittedTriangleFrontFace()](#rayquery-committedtrianglefrontface)\*\*The following table lists intrinsics available depending on the current current [COMMITTED_STATUS](#committed_status) (i.e. what type of hit has been commited, if any?). This applies regardless of whether [RayQuery::Proceed()](#rayquery-proceed) has returned `TRUE` (shader evaluation needed for traversal), or `FALSE` (traversal complete). If `TRUE`, additional methods than shown below are available based on the table above.

**Intrinsic** \ [CommittedStatus()](#rayquery-committedstatus)` COMMITTED_TRIANGLE_HIT``COMMITTED_PROCEDURAL_PRIMITIVE_HIT``COMMITTED_NOTHING `[TraceRayInline()](#rayquery-tracerayinline)\*\*\*`COMMITTED_STATUS` [CommittedStatus()](#rayquery-committedstatus)\*\*\*_**Ray system values:**\_uint [RayFlags()](#rayquery-rayflags)\*\*\*float3 [WorldRayOrigin()](#rayquery-worldrayorigin)\*\*\*float3 [WorldRayDirection()](#rayquery-worldraydirection)\*\*\*float [RayTMin()](#rayquery-raytmin)\*\*\*float [CommittedRayT()](#rayquery-committedrayt)\*\*\*_**Primitive/object space system values:**_uint [CommittedInstanceIndex()](#rayquery-committedinstanceindex)\*\*uint [CommittedInstanceID()](#rayquery-committedinstanceid)\*\*uint [CommittedInstanceContributionToHitGroupIndex()](#rayquery-committedinstancecontributiontohitgroupindex)\*\*uint [CommittedGeometryIndex()](#rayquery-committedgeometryindex)\*\*uint [CommittedPrimitiveIndex()](#rayquery-committedprimitiveindex)\*\*float3 [CommittedObjectRayOrigin()](#rayquery-committedobjectrayorigin)\*\*float3 [CommittedObjectRayDirection()](#rayquery-committedobjectraydirection)\*\*float3x4 [CommittedObjectToWorld3x4()](#rayquery-committedobjecttoworld3x4)\*\*float4x3 [CommittedObjectToWorld4x3()](#rayquery-committedobjecttoworld4x3)\*\*float3x4 [CommittedWorldToObject3x4()](#rayquery-committedworldtoobject3x4)\*\*float4x3 [CommittedWorldToObject4x3()](#rayquery-committedworldtoobject4x3)\*\*_**Hit specific system values:**\_float2 [CommittedTriangleBarycentrics()](#rayquery-committedtrianglebarycentrics)\*bool [CommittedTriangleFrontFace()](#rayquery-committedtrianglefrontface)\*---

#### RayQuery enums

---

##### COMMITTED_STATUS

```C++
enum COMMITTED_STATUS : uint
{
    COMMITTED_NOTHING,
    COMMITTED_TRIANGLE_HIT,
    COMMITTED_PROCEDURAL_PRIMITIVE_HIT
};

```

Return value for [RayQuery::CommittedStatus()](#rayquery-committedstatus).

ValueDefinition`COMMITTED_NOTHING`No hits have been committed yet.`COMMITTED_TRIANGLE_HIT`Closest hit so far is a triangle, a result of either the shader previously calling [RayQuery::CommitNonOpaqueTriangleHit()](#rayquery-commitnonopaquetrianglehit) or a fixed function opaque triangle intersection.`COMMITTED_PROCEDURAL_PRIMITIVE_HIT`Closest hit so far is a procedural primitive, a result of the shader previously calling [RayQuery::CommittProceduralPrimitiveHit()](#rayquery-commitproceduralprimitivehit).---

##### CANDIDATE_TYPE

```C++
enum CANDIDATE_TYPE : uint
{
    CANDIDATE_NON_OPAQUE_TRIANGLE,
    CANDIDATE_PROCEDURAL_PRIMITIVE
};

```

Return value for [RayQuery::CandidateType()](#rayquery-candidatetype).

ValueDefinition`CANDIDATE_NON_OPAQUE_TRIANGLE`Acceleration structure traversal has encountered a non opaque triangle (that would be the closest hit so far if committed) for the shader to evaluate. If the shader decides this is opaque, it needs to call [RayQuery::CommitNonOpaqueTriangleHit()](#rayquery-commitnonopaquetrianglehit).`CANDIDATE_PROCEDURAL_PRIMITIVE`Acceleration structure traversal has encountered a procedural primitive for the shader to evaluate. It is up to the shader to calculate all possible intersections for this procedural primitive and commit at most one hit that it sees would be the closest so far, via [RayQuery::CommitProceduralPrimitiveHit()](#rayquery-commitproceduralprimitivehit).---

#### RayQuery TraceRayInline

インラインレイトレースのパラメータを acceleration structure に初期化し、他のシェーダを呼び出さないようにします。 実際の走査はまだ開始しない。 呼び出したシェーダは、返された RayQuery オブジェクトを介してクエリの進捗を管理します。

概要についてはインラインレイシングを、詳細については TraceRayInline() コントロールフローダイアグラムを参照してください。 シュードコードの例はこちらです。

この組込み関数定義は、次の関数と同等です。

```C++
void RayQuery::TraceRayInline(
    RaytracingAccelerationStructure AccelerationStructure,
    uint RayFlags,
    uint InstanceInclusionMask,
    RayDesc Ray);
```

ParameterDefinition`RaytracingAccelerationStructure AccelerationStructure`Top-level acceleration structure to use. Specifying a NULL acceleration structure forces a miss.`uint RayFlags`Valid combination of [Ray flags](#ray-flags). Only defined ray flags are propagated by the system, e.g. visible to the [RayFlags()](#rayflags) shader intrinsic. These flags are OR'd with the [RayQuery](#rayquery)'s ray flags, and the combination must be valid (see the definition of each flag).`uint InstanceInclusionMask`<p>Bottom 8 bits of InstanceInclusionMask are used to include/reject geometry instances based on the InstanceMask in each [instance](#d3d12_raytracing_instance_desc):</p><p>`if(!((InstanceInclusionMask & InstanceMask) & 0xff)) { ignore intersection }`</p>`RayDesc Ray`[Ray](#ray-description-structure) to be traced. See [Ray-extents](#ray-extents) for bounds on valid ray parameters.`Return: RayQuery`Opaque object defining the state of a ray trace operation with no shader invocations. The caller calls methods on this object to participate in the acceleration structure traversal for as long as desired or until it ends. See [RayQuery](#rayquery).[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

##### TraceRayInline examples

- [Example 1](#tracerayinline-example-1): Trivially get a simple hit/miss from tracing a ray.
- [Example 2](#tracerayinline-example-2): More general case, handling all the possible states.
- [Example 3](#tracerayinline-example-3): Expensive scenario with simultaneous traces.

---

###### TraceRayInline example 1

(例のリストへ戻る)

些細な例ですが、レイのトレースから単純な三角形のヒット/ミスを取得するだけです。

この例と一致する、舞台裏で起こることを説明する図があります。特殊な TraceRayInline のコントロールフローです。

```C++
RaytracingAccelerationStructure myAccelerationStructure : register(t3);

float4 MyPixelShader(float2 uv : TEXCOORD) : SV_Target0
{
    ...
    // Instantiate ray query object.
    // Template parameter allows driver to generate a specialized
    // implementation.
    RayQuery<RAY_FLAG_CULL_NON_OPAQUE |
             RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES |
             RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH> q;

    // Set up a trace.  No work is done yet.
    q.TraceRayInline(
        myAccelerationStructure,
        myRayFlags, // OR'd with flags above
        myInstanceMask,
        myRay);

    // Proceed() below is where behind-the-scenes traversal happens,
    // including the heaviest of any driver inlined code.
    // In this simplest of scenarios, Proceed() only needs
    // to be called once rather than a loop:
    // Based on the template specialization above,
    // traversal completion is guaranteed.
    q.Proceed();

    // Examine and act on the result of the traversal.
    // Was a hit committed?
    if(q.CommittedStatus()) == COMMITTED_TRIANGLE_HIT)
    {
        ShadeMyTriangleHit(
            q.CommittedInstanceIndex(),
            q.CommittedPrimitiveIndex(),
            q.CommittedGeometryIndex(),
            q.CommittedRayT(),
            q.CommittedTriangleBarycentrics(),
            q.CommittedTriangleFrontFace() );
    }
    else // COMMITTED_NOTHING
         // From template specialization,
         // COMMITTED_PROCEDURAL_PRIMITIVE can't happen.
    {
        // Do miss shading
        MyMissColorCalculation(
            q.WorldRayOrigin(),
            q.WorldRayDirection());
    }
    ...
}
```

---

###### TraceRayInline example 2

(例のリストへ戻る)

より一般的なケースで、すべての可能な状態を処理します。

この図は、このタイプのシナリオをサポートするための完全な制御フローを説明しています。TraceRayInline のコントロールフロー。

```C++
RaytracingAccelerationStructure myAccelerationStructure : register(t3);

struct MyCustomAttrIntersectionAttributes { float4 a; float3 b; }

[numthreads(64,1,1)]
void MyComputeShader(uint3 DTid : SV_DispatchThreadID)
{
    ...
    // Instantiate ray query object.
    // Template parameter allows driver to generate a specialized
    // implemenation.  No specialization in this example.
    RayQuery<RAY_FLAG_NONE> q;

    // Set up a trace
    q.TraceRayInline(
        myAccelerationStructure,
        myRayFlags,
        myInstanceMask,
        myRay);

    // Storage for procedural primitive hit attributes
    MyCustomIntersectionAttributes committedCustomAttribs;

    // Proceed() is where behind-the-scenes traversal happens,
    // including the heaviest of any driver inlined code.
    // Returns TRUE if there's a task for the shader to perform
    // as part of traversal
    while(q.Proceed())
    {
        switch(q.CandidateType())
        {
        case CANDIDATE_PROCEDURAL_PRIMITIVE:
        {
            float tHit;
            MyCustomIntersectionAttributes candidateAttribs;

            // For procedural primitives, opacity is handled manually -
            // if an intersection is determined to not be opaque, just don't consider it
            // as a candidate.
            while(MyProceduralIntersectionEnumerator(
                tHit,
                candidateAttribs,
                q.CandidateInstanceIndex(),
                q.CandidatePrimitiveIndex(),
                q.CandidatetGeometryIndex()))
            {
                if( (q.RayTMin() <= tHit) && (tHit <= q.CommittedRayT()) )
                {
                    if(q.CandidateProceduralPrimitiveNonOpaque() &&
                        !MyProceduralAlphaTestLogic(
                            tHit,
                            attribs,
                            q.CandidateInstanceIndex(),
                            q.CandidatePrimitiveIndex(),
                            q.CandidateGeometryIndex()))
                    {
                        continue; // non opaque
                    }

                    q.CommitProceduralPrimitiveHit(tHit);
                    committedCustomAttribs = candidateAttribs;
                }
            }
            break;
        }
        case CANDIDATE_NON_OPAQUE_TRIANGLE:
        {
            if( MyAlphaTestLogic(
                q.CandidateInstanceIndex(),
                q.CandidatePrimitiveIndex(),
                q.CandidateGeometryIndex(),
                q.CandidateTriangleRayT(),
                q.CandidateTriangleBarycentrics(),
                q.CandidateTriangleFrontFace() )
            {
                q.CommitNonOpaqueTriangleHit();
            }
            if(MyLogicSaysStopSearchingForSomeReason()) // not typically applicable
            {
                q.Abort(); // Stop traversing and next call to Proceed()
                           // will return FALSE.
                           // Post-traversal results will just be based
                           // on what has been encountered so far.
            }
            break;
        }
        }
    }
    switch(q.CommittedStatus())
    {
    case COMMITTED_TRIANGLE_HIT:
    {
        // Do hit shading
        ShadeMyTriangleHit(
            q.CommittedInstanceIndex(),
            q.CommittedPrimitiveIndex(),
            q.CommittedGeometryIndex(),
            q.CommittedRayT(),
            q.CommittedTriangleBarycentrics(),
            q.CommittedTriangleFrontFace() );
        break;
    }
    case COMMITTED_PROCEDURAL_PRIMITIVE_HIT:
    {
        // Do hit shading for procedural hit,
        // using manually saved hit attributes (customAttribs)
        ShadeMyProceduralPrimitiveHit(
            committedCustomAttribs,
            q.CommittedInstanceIndex(),
            q.CommittedPrimitiveIndex(),
            q.CommittedGeometryIndex(),
            q.CommittedRayT());
        break;
    }
    case COMMITTED_NOTHING:
    {
        // Do miss shading
        MyMissColorCalculation(
            q.WorldRayOrigin(),
            q.WorldRayDirection());
        break;
    }
    }
    ...
}
```

---

###### TraceRayInline example 3

(例のリストへ戻る)

複数のトレースを同時に行う高価なシナリオ。

```C++
RaytracingAccelerationStructure myAccelerationStructure : register(t3);

float4 MyPixelShader(float2 uv : TEXCOORD) : SV_Target0
{
    ...
    RayQuery<RAY_FLAG_FORCE_OPAQUE> rayQ1;
    rayQ1.TraceRayInline(
      myAccelerationStructure,
      RAY_FLAG_NONE,
      myInstanceMask,
      myRay1);

    RayQuery<RAY_FLAG_FORCE_OPAQUE> rayQ2;
    rayQ2.TraceRayInline(
      myAccelerationStructure,
      RAY_FLAG_NONE,
      myInstanceMask,
      myRay2);

    // traverse rayQ1 and rayQ2 state machines simultaneously for some reason

    ...

    // Assignment from another generates a reference:
    RayQuery<RAY_FLAG_FORCE_OPAQUE> rayQ3 = rayQ2; // template config must match
                                  // rayQ3 and rayQ2 are aliases of each other

    ...

    // Reusing rayQ1 for a new query:
    // This is the one part of this example that's more likely to be useful: tracing a ray
    // sequentially after a previous trace is finished (so just reusing it's state for convenience).
    // It's also fine to just declare a different query object, relying on the compiler
    // to notice the lifetimes of each don't overlap.
    rayQ1.TraceRayInline(
      myAccelerationStructure,
      RAY_FLAG_NONE,
      myInstanceMask,
      myRay1);
    // This could be done any time, no need to wait for the original query to be in some state
    // to be allowed to replace with a new trace.
}
```

---

#### RayQuery Proceed

```C++
bool RayQuery::Proceed();
```

This is the main worker function for an inline raytrace. While [RayQuery::TraceRayInline()](#rayquery-tracerayinline) initializes the parameters for a raytrace, `RayQuery::Proceed()` is the function that causes the system to search the acceleration structure for hit candidates.

候補が見つかった場合、不透明な三角形の場合には自動的に処理されますが、特に不透明でない三角形や手続き的プリミティブに遭遇した場合には、次に何をすべきかを決定するためにシェーダに戻ります（下記の `TRUE` を参照ください）。

ParameterDefinitionReturn: `bool`<p>`TRUE` means there is a candidate for a hit that requires shader consideration. Calling [RayQuery::CandidateType()](#rayquery-candidatetype) after this reveals what the situation is.</p><p>`FALSE` means that the search for hit candidates is finished. Once `RayQuery::Proceed()` has returned `FALSE` for a given [RayQuery::TraceRayInline()](#rayquery-tracerayinline), further calls to `RayQuery::Proceed()` will continue to return `FALSE` and do nothing, unless the shader chooses to issue a new [RayQuery::TraceRayInline()](#rayquery-tracerayinline) call on the `RayQuery` object.</p>For an inline raytrace to be at all useful, `RayQuery::Proceed()` must be called at least once, and typically called repeatedly until it returns `FALSE`, unless the shader chooses to just bail out of a search early for any reason.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery Abort

```C++
void RayQuery::Abort();
```

Force a ray query into the finished state. In particular, calls to [RayQuery::Proceed()](#rayquery-proceed) will return `FALSE`.

`RayQuery::Abort()` can be called any time after [RayQuery::TraceRayInline()](#rayquery-tracerayinline) has been called.

[RayQuery intrinsics](#rayquery-intrinsics) also illustrates when this is valid to call.

> This method is provided for convenience for telling an outer loop that calls `RayQuery::Proceed()` such as below to break out, from within some nested code within the loop. The shader can accomplish the same thing manually as well, effectively abandoning a search. Either way, [RayQuery::CommittedStatus()](#rayquery-committedstatus) can be called to inform how to do any final shading.

```C+_
    while(rayQ.Proceed()) // `FALSE` after Abort() below
                          // (or search actually ended)
    {
        ...
        {
            ...
            {
                rayQ.Abort();
            }
            ...
        }
        ...
    }
```

#### RayQuery CandidateType

```C++

CANDIDATE_TYPE RayQuery::CandidateType()

```

When [RayQuery::Proceed()](#rayquery-proceed) has returned `TRUE` and there is a candidate hit for the shader to evaluate, `RayQuery::CandidateType()` reveals what type of candidate it is.

ParameterDefinition`Return value: CANDIDATE_TYPE`See [CANDIDATE_TYPE](#candidate_type).[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateProceduralPrimitiveNonOpaque

```C++
bool RayQuery::CandidateProceduralPrimitiveNonOpaque()
```

When [RayQuery::CandidateType()](#rayquery-candidatetype) has returned `CANDIDATE_PROCEDURAL_PRIMITIVE`, this indicates whether the acceleration structure combined with ray flags consider this procedural primitive candidate to be non-opaque. The only action the system might have taken with this information is to cull opaque or non-opaque primitives if the shader requested it via [RAY_FLAGS](#ray-flags). If a candidate has been produced however, culling obviously did not apply. At this point it is completely up to the shader if it wants to act any further on this opaque/non-opaque information.

シェーダは、このメソッドを呼び出すことさえせず、acceleration structure／レイフラグが決めたことを無視して、プロシージャルのヒットに対して独自の不透明度決定を行うことを選択できます。

---

#### RayQuery CommitNonOpaqueTriangleHit

```C++
void RayQuery::CommitNonOpaqueTriangleHit()
```

When [RayQuery::CandidateType()](#rayquery-candidatetype) has returned `CANDIDATE_NON_OPAQUE_TRIANGLE` and the shader determines this hit location is actually opaque, it needs to call `RayQuery::CommitNonOpaqueTriangleHit()`. `RayQuery::CommitNonOpaqueTriangleHit()` can be called multiple times per non opaque triangle candidate - after the first call, subsequent calls simply have no effect for the current candidate.

The system will have already determined this candidate hit T would \> the ray's TMin and \< TMax ([RayQuery::CommittedRayT()](#rayquery-committedrayt)), so the shader need not manually verify these conditions.

Once committed, the system remembers hit properties like barycentrics, front/back facing of the hit, acceleration structure node information like PrimitiveIndex. As long as this remains the closest hit, [RayQuery::CommittedTriangleBarycentrics()](#rayquery-committedtrianglebarycentrics) etc. will return the properties for this hit.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommitProceduralPrimitiveHit

```C++
void RayQuery::CommitProceduralPrimitiveHit(float tHit)
```

When [RayQuery::CandidateType()](#rayquery-candidatetype) has returned `CANDIDATE_PROCEDURAL_PRIMITIVE`, the shader is responsible for procedurally finding all intersections for this candidate. If the shader is manually accumulating transparency, it can do so manually without the system knowing this. The one responsibility the system expects of the shader is that it must only commit hit(s) when it manually determines that a given hit being reported satisfies TMin \<= t \<= TMax ( [RayQuery::CommittedRayT()](#rayquery-committedrayt)). When the range condiditon is satisfied, the shader can validly commit a hit via `RayQuery::CommitProceduralPrimitiveHit()`. See [ray extents](#ray-extents) for some further discussion.

The shader must manually store (e.g. in local variables) any hit information that it might want later. The exception is data that the system tracks, such as the t value, or acceleration structure node information such as InstanceIndex, which can be retrieved via methods like [RayQuery::CommittedInstanceIndex()](#rayquery-committedinstanceindex) for whatever the current closest commited hit is.

`RayQuery::CommitProceduralPrimitiveHit()` を、与えられたプリミティブの候補を処理する間に複数回呼んでもかまいません。 この場合、呼び出されるたびに、シェーダはコミットされるヒットがコミットされた以前のヒットに基づく最新の[TMin ... TMax]（を含む）範囲内であることを確認する必要があります。 別の方法として、どのヒットが最も近い範囲にあるか（ある場合）手動で判断し、一度だけコミットすることもできます。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

ParameterDefinition`float tHit`t value for intersection that shader determined is closest so far and >= the ray's TMin.---

#### RayQuery CommittedStatus

```C++
COMMITTED_STATUS CommittedStatus()
```

Status of the closest hit that has been committed so far (if any). This can be called any time after [RayQuery::TraceRayInline()](#rayquery-tracerayinline) has been called.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

ParameterDefinition`Return value: COMMITTED_STATUS`See [COMMITTED_STATUS](#committed_status).---

#### RayQuery RayFlags

これは、現在のレイフラグ (テンプレートフラグとダイナミックフラグの ORd) を含む uint です。

```C++
uint RayQuery::RayFlags();
```

これは、例えば、手続き型プリミティブの処理中に、アプリが現在のレイのカリングフラグを見て、カスタム交差点コードに対応するカリングを適用したい場合に便利です。

> It is arguable that this method appears redundant given the shader could just remember what flags it originally passed into [RayQuery::TraceRayInline()](#rayquery-tracerayinline). The rationale is that since the system needs to keep this information anyway, no need for the app to waste a live register storing it, so in effect it can eliminate redundancy.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery WorldRayOrigin

現在のレイのワールドスペースの原点。

```C++
float3 RayQuery::WorldRayOrigin();
```

> It is arguable that this method appears redundant given the shader could just remember what flags it originally passed into [RayQuery::TraceRayInline()](#rayquery-tracerayinline). The rationale is that since the system needs to keep this information anyway, no need for the app to waste a live register storing it, so in effect it can eliminate redundancy.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery WorldRayDirection

現在のレイのワールド空間方向。

```C++
float3 RayQuery::WorldRayDirection();
```

> It is arguable that this method appears redundant given the shader could just remember what flags it originally passed into [RayQuery::TraceRayInline()](#rayquery-tracerayinline). The rationale is that since the system needs to keep this information anyway, no need for the app to waste a live register storing it, so in effect it can eliminate redundancy.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery RayTMin

レイのパラメトリックな始点を表す float です。

```C++
float RayQuery::RayTMin();
```

RayTMin defines the starting point of the ray according to the following
formula: `Origin + (Direction \* RayTMin)`. `Origin` and `Direction` may be in
either world or object space, which results in either a world or an
object space starting point.

RayTMin is defined when calling [RayQuery::TraceRayInline()](#rayquery-tracerayinline), and is constant
for the duration of that call.

> It is arguable that this method appears redundant given the shader could just remember what flags it originally passed into [RayQuery::TraceRayInline()](#rayquery-tracerayinline). The rationale is that since the system needs to keep this information anyway, no need for the app to waste a live register storing it, so in effect it can eliminate redundancy.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateTriangleRayT

ヒットを考慮する三角形の候補が存在するパラメトリックな距離を表す float です。

```C++
float RayQuery::CandidateTriangleRayT();
```

`CandidateTriangleRayT()` defines the candidate point for non-opaque triangles along the ray according to the
following formula: `Origin + (Direction * CandidateTriangleRayT)`. `Origin` and
`Direction` may be in either world or object space, which results in
either a world or an object space ending point.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedRayT

これまでのところ、最も近いヒットを約束したパラメトリック距離を表す float です。

```C++
float RayQuery::CommittedRayT();
```

`CommittedRayT()` defines the current TMax point along the ray according to the
following formula: `Origin + (Direction * CommittedRayT)`. `Origin` and
`Direction` may be in either world or object space, which results in
either a world or an object space ending point.

`CommittedRayT()` is initialized by the [RayQuery::TraceRayInline()](#rayquery-tracerayinline) call to
`RayDesc::TMax`, and updated during the trace query as new intersections are commited.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateInstanceIndex

現在のヒット候補のトップレベル構造における、現在のインスタンスの自動生成されたインデックスです。

```C++
uint RayQuery::CandidateInstanceIndex();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateInstanceID

ユーザー提供の `InstanceID` （現在のヒット候補の最上位構造体内の bottom-level acceleration structure インスタンス上）。

```C++
uint RayQuery::CandidateInstanceID();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateInstanceContributionToHitGroupIndex

現在のヒット候補のトップレベル構造体内の Bottom-Level Acceleration Structure インスタンスにあるユーザー提供の `InstanceContributionToHitGroupIndex` です。

```C++
uint RayQuery::CandidateInstanceContributionToHitGroupIndex();
```

インラインレイトレーシングでは、この値は、名前にヒットグループインデックスへの貢献を記述していても、機能的には何の効果もありません。 この名前はダイナミックシェーダーベースのレイトレーシングでの動作に言及するものです。これは、単に完全性のために RayQuery を介して利用可能になり、インライン レイトレーシングがアクセラレーション構造内のすべてを見ることができるようにな ります。

> An app might use this a way to store another arbitrary user value per instance into instance data.
> Or it might be sharing an acceleration structure between dynamic-shader-based and inline raytracing:
> In the dynamic-shader-based case, the value participates in [shader table indexing](#addressing-calculations-within-shader-tables)
> and in the inline case the app may achieve some similar equivalent effect manually indexing into its own data structures.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateGeometryIndex

現在のヒット候補の最下位レベルのアクセラレーション構造における現在のジオメトリの自動生成されたインデックスです。

```C++
uint RayQuery::CandidateGeometryIndex();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidatePrimitiveIndex

現在のヒット候補の Bottom-Level Acceleration Structure インスタンス内のジオメトリ内のプリミティブの自動生成されたインデックス。

```C++
uint RayQuery::CandidatePrimitiveIndex();
```

`D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES` の場合、これはジオメトリオブジェクト内の三角形のインデックスになります。

`D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS` では、ジオメトリ オブジェクトを定義する AABB 配列へのインデックスになります。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateObjectRayOrigin

現在のレイのオブジェクト空間原点。Object-space は、現在のヒット候補の bottom-level acceleration structure の空間を指します。

```C++
float3 RayQuery::CandidateObjectRayOrigin();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateObjectRayDirection

現在のレイのオブジェクト空間方向。オブジェクト空間は、現在のヒット候補の現在の bottom-level acceleration structure の空間を参照する。

```C++
float3 RayQuery::CandidateObjectRayDirection();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateObjectToWorld3x4

オブジェクト空間からワールド空間への変換のための行列。オブジェクト空間は、現在のヒット候補に対する現在の bottom-level acceleration structure の空間を指します。

```C++
float3x4 RayQuery::CandidateObjectToWorld3x4();
```

`CandidateObjectToWorld4x3()` との唯一の違いは、行列が転置されることです -- どちらか都合のよいほうを使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateObjectToWorld4x3

オブジェクト空間からワールド空間への変換のための行列。オブジェクト空間は、現在のヒット候補に対する現在の bottom-level acceleration structure の空間を指します。

```C++
float4x3 RayQuery::CandidateObjectToWorld4x3();
```

`CandidateObjectToWorld3x4()` との唯一の違いは、行列が転置されることです -- どちらか都合の良いほうを使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateWorldToObject3x4

ワールド空間からオブジェクト空間への変換を行うための行列。オブジェクト空間とは、現在のヒット候補の bottom-level acceleration structure の空間を指します。

```C++
float3x4 RayQuery::CandidateWorldToObject3x4();
```

`CandidateWorldToObject4x3()` との唯一の違いは、行列が転置されることです -- どちらか都合の良い方を使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateWorldToObject4x3

ワールド空間からオブジェクト空間への変換を行うための行列。オブジェクト空間とは、現在のヒット候補の bottom-level acceleration structure の空間を指します。

```C++
float3x4 RayQuery::CandidateWorldToObject4x3();
```

`CandidateWorldToObject3x4()` との唯一の違いは、この行列が転置されていることです -- どちらか都合の良い方を使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedInstanceIndex

これまでにコミットされた最も近いヒットに対するトップレベル構造体のインスタンスの自動生成されたインデックスです。

```C++
uint RayQuery::CommittedInstanceIndex();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedInstanceID

ユーザが提供する `InstanceID` (top-level acceleration structure のインスタンス) です。

```C++
uint RayQuery::CommittedInstanceID();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedInstanceContributionToHitGroupIndex

これまでにコミットされた最も近いヒットのトップレベル構造内の Bottom-Level Acceleration Structure インスタンス上のユーザー提供の `InstanceContributionToHitGroupIndex` 。

```C++
uint RayQuery::CommittedInstanceContributionToHitGroupIndex();
```

インラインレイトレーシングでは、この値は、名前にヒットグループインデックスへの貢献を記述していても、機能的には何の効果もありません。 この名前はダイナミックシェーダーベースのレイトレーシングでの動作に言及するものです。これは、単に完全性のために RayQuery を介して利用可能になり、インライン レイトレーシングがアクセラレーション構造内のすべてを見ることができるようにな ります。

> An app might use this a way to store another arbitrary user value per instance into instance data.
> Or it might be sharing an acceleration structure between dynamic-shader-based and inline raytracing:
> In the dynamic-shader-based case, the value participates in [shader table indexing](#addressing-calculations-within-shader-tables)
> and in the inline case the app may implement some similar equivalent effect manually indexing into its own data structures.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedGeometryIndex

これまでにコミットされた最も近いヒットのための Bottom-Level Acceleration Structure 内のジオメトリの自動生成されたインデックスです。

```C++
uint RayQuery::CommittedGeometryIndex();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedPrimitiveIndex

これまでにコミットされた最も近いヒットのための Bottom-Level Acceleration Structure インスタンス内のジオメトリ内のプリミティブの自動生成されたインデックス。

```C++
uint RayQuery::CommittedPrimitiveIndex();
```

`D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES` の場合、これはジオメトリオブジェクト内の三角形のインデックスになります。

`D3D12_RAYTRACING_GEOMETRY_TYPE_PROCEDURAL_PRIMITIVE_AABBS` では、ジオメトリ オブジェクトを定義する AABB 配列へのインデックスになります。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedObjectRayOrigin

レイのオブジェクト空間原点。オブジェクト空間は、これまでにコミットされた最も近いヒットのための bottom-level acceleration structure の空間を指します。

```C++
float3 RayQuery::CommittedObjectRayOrigin();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedObjectRayDirection

レイのオブジェクト空間方向。オブジェクト空間は、これまでに行われた最も近いヒットのための bottom-level acceleration structure の空間を参照します。

```C++
float3 CommittedObjectRayDirection();
```

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedObjectToWorld3x4

オブジェクト空間から世界空間に変換するための行列。オブジェクト空間は、これまでに行われた最も近いヒットのための bottom-level acceleration structure の空間を参照します。

```C++
float3x4 RayQuery::CommittedObjectToWorld3x4();
```

`CommittedObjectToWorld4x3()` との唯一の違いは、行列が転置されることです -- どちらか都合の良い方を使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedObjectToWorld4x3

オブジェクト空間から世界空間に変換するための行列。オブジェクト空間は、これまでに行われた最も近いヒットのための bottom-level acceleration structure の空間を参照します。

```C++
float4x3 RayQuery::CommittedObjectToWorld4x3();
```

`CommittedObjectToWorld3x4()` との唯一の違いは、行列が転置されることです -- どちらか都合の良い方を使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedWorldToObject3x4

ワールド空間からオブジェクト空間への変換を行うための行列。オブジェクト空間とは、これまでにコミットされた最も近いヒットに対する最下層 acceleration structure の空間を指します。

```C++
float3x4 RayQuery::CommittedWorldToObject3x4();
```

`CommittedWorldToObject4x3()` との唯一の違いは、行列が転置されることです -- どちらか都合の良い方を使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CommittedWorldToObject4x3

ワールド空間からオブジェクト空間への変換を行うための行列。オブジェクト空間とは、これまでにコミットされた最も近いヒットに対する最下層 acceleration structure の空間を指します。

```C++
float3x4 RayQuery::CommittedWorldToObject4x3();
```

`CommittedWorldToObject3x4()` との唯一の違いは、行列が転置されることです -- どちらか都合の良い方を使用してください。

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

---

#### RayQuery CandidateTriangleBarycentrics

```C++
float2 RayQuery::CandidateTriangleBarycentrics
```

Applicable for the current hit candidate, if [RayQuery::CandidateType()](#rayquery-candidatetype) returns `CANDIDATE_NON_OPAQUE_TRIANGLE`. Returns hit location barycentric coordinates.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

ParameterDefinition`Return value: float2`<p>Given attributes `a0`, `a1` and `a2` for the 3 vertices of a triangle, `float2.x` is the weight for `a1` and `float2.y` is the weight for `a2`. For example, the app can interpolate by doing: `a = a0 + float2.x \* (a1-a0) + float2.y\* (a2 -- a0)`.</p>#### RayQuery CandidateTriangleFrontFace

```C++
bool RayQuery::CandidateTriangleFrontFace
```

Applicable for the current hit candidate, if [RayQuery::CandidateType()](#rayquery-candidatetype) returns `CANDIDATE_NON_OPAQUE_TRIANGLE`. Reveals whether hit triangle is front of back facing.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

ParameterDefinition`Return value: bool`<p>`TRUE` means front face, `FALSE` means back face.</p>---

#### RayQuery CommittedTriangleBarycentrics

```C++
float2 RayQuery::CommittedTriangleBarycentrics
```

Applicable for the closest committed hit so far, if that hit is a triangle (which can be determined from [RayQuery::CommittedStatus()](#rayquery-committedstatus). Returns hit location barycentric coordinates.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

ParameterDefinition`Return value: float2`<p>Given attributes `a0`, `a1` and `a2` for the 3 vertices of a triangle, `float2.x` is the weight for `a1` and `float2.y` is the weight for `a2`. For example, the app can interpolate by doing: `a = a0 + float2.x \* (a1-a0) + float2.y\* (a2 -- a0)`.</p>---

#### RayQuery CommittedTriangleFrontFace

```C++
bool RayQuery::CommittedTriangleFrontFace
```

Applicable for the closest committed hit so far, if that hit is a triangle (which can be determined from [RayQuery::CommittedStatus()](#rayquery-committedstatus). Reveals whether hit triangle is front of back facing.

[RayQuery intrinsics](#rayquery-intrinsics) illustrates when this is valid to call.

ParameterDefinition`Return value: bool`<p>`TRUE` means front face, `FALSE` means back face.</p>---

## Payload access qualifiers

シェーダモデル 6.6 と 6.7 はレイペイロード構造にペイロードアクセス修飾子 (PAQs) を追加しました。PAQ はペイロードフィールドの読み込みと書き込みのセマンティクスを記述する注釈で、つまり、どのシェーダステージが与えられたフィールドを読み込み、または書き込むかを記述します。追加されたセマンティック情報は、実装がレジスタの圧力を減らすのに役立ち、ペイロードの状態がメモリに流出するのを避けることができます。このため、各ペイロードフィールドに対して、可能な限り狭い修飾子を使用することが奨励されます。

---

### Availability

シェーダモデル 6.6 以前では、ペイロードアクセス修飾子 (PAQ) はサポートされていません。

SM 6.6 では、PAQ はデフォルトで無効化されています。ユーザーは、DXC の `enable-payload-qualifiers` コマンドラインフラグを使用して、この機能にオプトインすることができます。

SM 6.7 以降では、PAQ はデフォルトで有効になっています。ユーザーは DXC の `disable-payload-qualifiers` コマンドラインフラグを使用して、この機能をオプトアウトすることができます。

同じライブラリ/ステートオブジェクト内で、アノテーションされたペイロードとアノテーションされていないペイロードを混在させることは合法です。

---

### Payload size

ペイロードアクセス修飾子（PAQs）を使用すると、D3D12_RAYTRACING_SHADER_CONFIG の `MaxPayloadSizeInBytes` プロパティは不要になりました。 このフィールドは、SM 6.7 以降で PAQ が有効（上記によるデフォルト）である場合、ドライバによって無視されます。

SM 6.7 以降で PAQ が無効な場合、 `MaxPayloadSizeInBytes` が引き続き使用されます。

SM 6.6 では、ドライバ実装者の移行を容易にするために、PAQ の使用有無にかかわらず、アプリケーションは `MaxPayloadSizeInBytes` を設定しなければなりません。

---

### Syntax

ペイロードとして使用される構造体タイプは、以下のように型宣言の一部として `[raypayload]` type 属性を持たなければなりません。

```C++
struct [raypayload] MyPayload{ ... };
```

上記のようにペイロードとしてマークされた構造体のみが TraceRay 呼び出しの引数として、また closehit、anyhit または miss シェーダのペイロードパラメータとして使用することができま す。

ペイロード型は、各メンバー変数に PAQ を付ける必要があります。ペイロードアクセス修飾子（PAQ）の構文は、リソースバインディングの構文に従います。有効なアノテーションはこの構文に従います。

```C++
Type Field : qualifier1([s0, s1, ...]) : qualifier2([s0, s1, ...]);
```

有効な修飾子は `read` と `write` の 2 つです。それぞれの修飾子は、それが適用されるシェーダステージ（上記の定義では `s0...sN` ）を含む、引数リストを運びます。有効なシェーダステージは以下の通りです。 `anyhit` , `closesthit` , `miss` , `caller` 。

各フィールドは `read` と `write` の修飾子を 1 つずつ宣言する必要があります。修飾子の引数リストは空にすることができます。

PAQ は、スカラー、配列、構造体、ベクトル、または行列の型に対してのみ指定することができます。他の型を含むペイロード型（すなわち PAQ を持つ構造体）の場合、PAQ はネストした型の中で指定する必要があり、メンバに直接アノテーションを付けることは許可されません。

---

### Semantics

一般的に、 `read` 修飾子はシェーダステージがペイロードフィールドを読み取ることを示し、 `write` 修飾子はシェーダステージがそのフィールドに書き込むことを示すと言われています。

アプリケーション開発者のために、以下の分類は、ほとんどのシナリオでメンタルモデルとして十分であるべき修飾子の動作の説明を提供します。より正確なセマンティクスの定義については、次のセクションを参照してください。

**anyhit/closesthit/miss stages**

qualifiersemantic`read`<p>Indicates that for the given stage, the payload field is available for reading and will contain the last value written by a previous stage.</p>`write`<p>Indicates that the payload field will be overwritten by the given stage, if the stage executes. The field will be overwritten regardless of whether the executed shader explicitly assigns a value. If a value is not explicitly written by the shader, then an undefined value is written to the payload field at the end of the shader.</p><p>If the given stage does not execute, the payload field remains unmodified. A shader stage may not execute for various reasons. Examples include [ray flags](#ray-flags) (`FORCE_OPAQUE` does not execute anyhit, `SKIP_CLOSESTHIT` does not execute closesthit) or dynamic behavior (rays that miss all geometry do not execute closeshit, rays that hit geometry do not execute miss, etc).</p><p>_Note that invoking a NULL shader identifier in the Shader Table is equivalent to executing an empty shader, so in that case the stage counts as executed._</p>`read+write`<p>Indicates that the given stage may read and/or write the payload field. This is analogous to inout function parameters. If the shader does not explicitly assign a value or if the stage is not executed, the payload field remains unmodified.</p>`<none>`<p>The payload field is neither available for reading nor are its contents modified by the given stage.</p>**caller stage**

`caller` ステージは、TraceRay 関数の呼び出し元（つまり、最も一般的にはレイ生成シェーダ）を表します。TraceRay への「入力」を表すフィールド（例えば、ランダムシード）は、最初に呼び出し元によって書き込まれなければならないので、そのようなフィールドのペイロード アクセス修飾子 (PAQ) には `write(caller)` を含める必要があります。TraceRay からの「出力」を表すフィールド（例えば、出力色）は、呼び出し元によって読み込まれなければならないので、そのようなフィールドの PAQ は `read(caller)` を含まなければなりません。 `read(caller)` と `write(caller)` の両方を指定することにより、フィールドは TraceRay への入力と出力の両方になることが可能です。

---

### Detailed semantics

ペイロードアクセス修飾子（PAQ）は、シェーダステージ間の遷移で有効になります。シェーダステージの遷移は、TraceRay の呼び出し時や戻り時、あるいは anyhit/closesthit/miss シェーダに入ったり出たりする時に発生します。概念的には、引数としてペイロードを受け取る各シェーダステージは、レイに添付された実際のペイロードのローカル作業コピーとしてペイロードパラメータを作成します。そして、PAQ は、シェーダステージに入るときと出るときに、ローカルコピーと実際のペイロードの間でどのフィールドがコピーされるかを決定します。

具体的には、以下のセマンティクスが適用されます。

- ペイロードを受け取るシェーダステージの実行開始時に、そのステージで `read` とマークされているフィールドは、実際のペイロードからシェーダが受け取るパラメータにコピーされます。入力パラメータの他のすべてのフィールドは未定義のままです。
- ペイロードを受け取るシェーダステージの実行終了時に、そのステージで `write` とマークされているフィールドは、シェーダのパラメータから実際のペイロードにコピーバックされます。パラメータの他のフィールドに書き込まれた値はすべて無視されます。
- TraceRay 呼び出しの実行終了時に、 `read(caller)` とマークされたペイロード型のフィールドは、実際のペイロードから TraceRay に渡されたペイロード引数にコピーバックされます。ペイロード引数の他の全てのフィールドは未定義の内容を持つ。
- `read` ステージは必ず `write` ステージに先行しなければなりません。

実装は実際のペイロードをどのように構成してもよく、明確に定義された値を持ち、将来読み込まれる可能性のあるペイロードフィールドの値のみを保持する必要があります。

---

#### Local working copy

ペイロード アクセス修飾子（PAQ）はステージ遷移にのみ影響するため、コンパイラは、シェーダーがペイロードのローカル作業コピーにアクセスする方法を制限しません。このため、ペイロード型の変数を自由に代入することができます。特に、関数の引数としてペイロードを渡すことも可能である（この場合、通常の構造体パラメータと同様に in/out セマンティクスが適用される）。

シェーダがペイロードフィールドにアクセスする際に、指定された PAQ を守り、望ましい動作を実現することは、開発者の責任です。開発者が意図しない影響を受けないように、コンパイラは未定義の値や無視される書き込みにつながる可能性のあるアクセスが検出されたときはいつでも警告を出そうとします。

---

#### Shader stage sequence

関連するシェーダステージの一般的なシーケンスは次のとおりです。(交差点シェーダはペイロードにアクセスできないので省かれています)。

```
caller -> anyhit -> (closesthit|miss) -> caller
           ^  |
           |__|
```

TraceRay 操作の過程で複数の anyhit シェーダを実行できるため、'anyhit' はそれ自身に先行し成功します。

A [TraceRay](#traceray) call may not invoke any shaders at all (e.g. a ray that hits triangle geometry marked as opaque, while also specifying the `SKIP_CLOSESTHIT` [ray flag](#rayflags)). In that case, [payload access qualifiers](#payload-access-qualifiers) still apply to the `caller -> caller` transition.

ペイロード宣言時に、コンパイラによって以下のルールが強制されます。

1.  `write` ステージはすべて `read` ステージに続いていなければなりません。
2.  呼び出し側は TraceRay を呼び出す前にフィールドを初期化する必要があるか `caller` を `write` に追加します。

---

### Example

ペイロードアクセス修飾子を持つペイロード構造体の例を以下に示す。

```C++
struct Nested { float a, b, c; }

struct [raypayload] MyPayload
{
    // "Pure" output from closesthit or miss:
    float4 irradiance : read(caller) : write(closesthit, miss);

    // "Pure" input into all stages, not preserved over TraceRay
    uint seed : read(anyhit,closesthit,miss) : write(caller);

    // Simple input into all stages, preserved over TraceRay
    uint seed : read(anyhit,closesthit,miss,caller) : write(caller);

    // In-out flag, overwritten by miss (e.g. for shadow/visibility):
    bool hasHit : read(caller) : write(caller,miss);

    // Anyhits communicating amongst themselves, with initialization from caller
    float a : read(anyhit) : write(caller,anyhit);

    // Nested struct which itself does not have PAQs:
    Nested n : read(caller) : write(closesthit,miss);

    // Nested payload struct which itself has PAQs:
    MyBasePayload base;
};

```

### Guidelines

ペイロードアクセス修飾子の定義を正しく指定するためのガイドラインを以下に示す。

1. TraceRay を呼び出した後、呼び出し元は返されたフィールドを使用しますか（ループのようなケースも含めて）？ `read` に `caller` を追加してください。
2. anyhit/closesthit/miss ステージのシェーダはフィールドを読みますが、同じステージのシェーダはフィールドを書き込むことはありませんか？対応するシェーダステージを `read` に追加しますが、 `write` には追加しません。
3. anyhit/closesthit/miss ステージのすべてのシェーダが無条件にフィールドを書き込み、決して読み込まないのですか？対応するシェーダステージを `write` に追加し、 `read` には追加しないでください。
4. anyhit/closesthit/miss シェーダが条件付きでフィールドを変更するか、あるステージのシェーダがフィールドを書き、他のシェーダが書き込まないか？ `write` をすべてのシェーダで無条件にするようにし、ガイドライン (4) を適用してみてください。それが不可能な場合は、 `read` と `write` の両方にステージを追加してください。
5. パフォーマンスを最大化するために、できるだけ少ない修飾子/ステージを指定する。可能な限り、フィールドを「純粋な入力」または「純粋な出力」（後の例を参照）にするようにします。
6. パラメータリストにはペイロードはありません。

---

### Optimization potential

**1) Shortened lifetimes**

```C++
struct [raypayload] MyPayload
{
    float ahInput : write(caller) : read(anyhit);
};
```

この例では、 `ahInput` は anyhit ステージ以降にアクセスされない、つまり anyhit の後にそのライフタイムが終了する。したがって、実装は、後続のステージのためにこのフィールドを自由に無視することができ、closehit および miss シェーダにおけるシェーダレジスタの圧力を低減することができます。

**2) Disjoint lifetimes**

```C++
struct [raypayload] MyPayload
{
    float alpha : write(caller,anyhit) : read(closesthit);
    bool didHit : write(closesthit, miss) : read(caller);
    ...
};
```

この例では、 `alpha` のライフタイムは、closehit シェーダが値を読んだ時点で終了し、 `alpha` に割り当てられたリソース（例えば、レジスタ）は自由に再利用できるようになります。 `alpha` と `didHit` の寿命は現在、証明可能に分離しているので、実装は `alpha` のレジスタを再利用して `didHit` を TraceRay の呼び出し元に伝播することが許されています。

---

### Advanced examples

---

#### Various accesses and recursive TraceRay

```C++

struct [raypayload] Payload
{
    float a : write(closesthit, miss) : read(caller);
    float b : write(miss) : read(caller);
    float c : write(caller, closesthit) : read(caller, closesthit);
    float d : write(caller) : read(closesthit, miss);
};

[shader("closesthit")]
void ClosestHit(inout Payload payload)
{
    float tmp1 = payload.a; // WARNING: reading undefined value ('a' not read(closesthit))
    float tmp2 = payload.b; // WARNING: reading undefined value ('b' not read(closesthit))
    float tmp3 = payload.c;
    float tmp4 = payload.d;

    payload.a = 3;
    payload.b = 3; // WARNING: write will be ignored after CH returns ('b' not write(closesthit))
    payload.c = 3;
    payload.d = 3; // WARNING: write will be ignored after CH returns ('d' not write(closesthit))

    Payload p;
    p.a = 3; // WARNING: value will be undefined inside TraceRay ('a' not write(caller))
    p.b = 3; // WARNING: value will be undefined inside TraceRay ('b' not write(caller))
    p.c = 3;
    p.d = 3;
    TraceRay(p);
    float tmp5 = p.a;
    float tmp6 = p.b;
    float tmp7 = p.c;
    float tmp8 = p.d; // WARNING: reading undefined value ('d' not read(caller))

    Payload p2 = payload; // copying entire payload is OK, but inherits any potential undefined values
    TraceRay(p2);
}

```

---

#### Payload as function parameter

```C++
struct [raypayload] MyPayload
{
    float a : write(caller, closesthit, miss) : read(caller);
};

// could also use plain 'in' or 'out', behaves like normal struct parameter
void foo(inout MyPayload p)
{
    p.a = 123;
}

[shader(“closesthit”)]
void ClosestHit(inout MyPayload p)
{
    foo(p);
}
```

---

#### Forwarding payloads to recursive TraceRay calls

```C++
struct [raypayload] MyPayload
{
    float pureInput : write(caller) : read(closesthit);
    float pureOutput : write(closesthit) : read(caller);
};

[shader(“closesthit”)]
void ClosestHit(inout MyPayload p)
{
    // undefined contents due to lack of read(closesthit) and write(caller)
    float a = p.pureOutput;

    // This write will not be visible inside recursive closesthit stages, due to
    // the lack of write(caller) and read(closesthit) annotations. Once the recursive
    // TraceRay call returns, the value of p.pureOutput will be whatever the recursive
    // closesthit invocation assigned, or undefined if the recursive invocation did
    // not assign anything.
    p.pureOutput = 222;

    // This write will be visible inside the recursive closesthit stages, thanks to
    // write(caller). We are operating on the local copy of the payload here, so the
    // fact that p.pureInput does not specify write(closesthit) is irrelevant for this
    // point (it only matters at the stage transition out of closeshit).
    p.pureInput = 123;
    TraceRay(p);

    float c = p.pureInput; // now undefined due to lack of read(caller)
    float d = p.pureOutput; // result of recursive TraceRay, or undefined
                            // if the recursion didn't write

    p.pureOutput = 444; // visible to the caller
}
```

---

#### Pure input in a loop

```C++
struct [raypayload] MyPayload
{
    float pureInput : write(caller) : read(closesthit);
};

[shader(“raygeneration”)]
void Raygen()
{
    MyPayload p;

    p.pureInput = 123;

    while (condition)
    {
      // p.pureInput will be undefined after the first
      // loop iteration. Either specify read(caller)
      // to preserve the value or manually keep a copy
      // of the value live across the TraceRay call.
      TraceRay(p);
    }
}
```

---

#### Conditional pure output overwriting initial value

```C++
struct [raypayload] MyPayload
{
    float pureOutput : write(caller,closesthit) : read(caller);
};

[shader(“closesthit”)]
void ClosestHit(inout MyPayload p)
{
    if( condition )
        p.pureOutput = 123;
    else
    {
        // p.pureOutput becomes undefined because we do not
        // write it explicitly and omit read(closeshit)
    }
}
```

---

### Payload access qualifiers in DXIL

Payload Access Qualifiers (PAQ) は、DXIL では型アノテーションのようにメタデータとして表現されます。 `dx.dxrPayloadAnnotations` メタデータノードは、タグ `kDxilPayloadAnnotationStructTag(0)` と、ペイロードタイプの undef 値とメタデータノードの参照からなるペアのリストを含む単一のノードを参照しています。

```C++
!dx.dxrPayloadAnnotations = !{!24}
|--->!24 = !{i32 0, %struct.Payload undef, !25, %struct.MyColor undef, !29}
     |--->!25 = !{!26, !27, !28}
     |     |--->!26 = !{i32 0, i32 0}
     |     |--->!27 = !{i32 0, i32 819}
     |     |--->!28 = !{i32 0, i32 51}
     |--->!29 = !{!30, !30, !30}
           |--->!30 = !{i32 0, i32 545}
```

したがって、ペイロード タイプのすべてのフィールドは 1 つのメタデータ ノートで表現され、その順序はタイプのレイアウトと一致していなければならない。

フィールドごとのメタデータは、 `kDxilPayloadFieldAnnotationAccessTag(0)` に続いて、このフィールドの PAQ を格納するビットマスクを含んでいます。各シェーダステージで 4 ビットがビットマスクに使用されます。ビット 0-1 が PAQ に使用されます。ビット 0 はそのシェーダステージがリードアクセスであることを示し、ビット 1 はライトアクセスであることを示しています。

各ステージのビットは次のとおりです。

StageBitsCaller0-3Closesthit4-7Miss8-11Anyhit12-15 注：他のペイロードタイプで宣言されたフィールドは、どのビットも設定されてはいけません。ペイロードアクセス情報は、フィールドではなくペイロードタイプから取得する必要があります。

---

# DDI

---

## General notes

---

### Descriptor handle encodings

アプリケーション定義データを含むシェーダーテーブルは、ローカルルートシグネチャに存在するとき、8 バイトの GPU ディスクリプタハンドルを保持することができることを思い出してください。

記述子ハンドルは、ドライバ定義の記述子インクリメントを記述子ヒープベースアドレス（これもドライバからアプリに報告される）に追加することによって、アプリによって生成されます。記述子ハンドルは、すべての記述子ヒープでグローバルに一意であることが必要です（記述子ヒープタイプに関係なく）。これにより、ツールやデバッグ層は、シェーダーテーブルにディスクリプタを配置するときに、アプリケーションの意図を安価に理解することができます。たとえば、シェーダーテーブルによって参照されるディスクリプタが、コマンドリストで現在バインドされているディスクリプタヒープから来ることを検証できます（以前に使用されたディスクリプタヒープを指す古いディスクリプタハンドルや、無効なディスクリプタとは対照的です）。

---

## State object DDIs

概要については、「状態オブジェクト」を参照してください。DDI は一般的に API をミラーリングします（そして、D3D12 の他の部分の DDI パターンに従います）。注目すべき例外は以下に呼び出されます。

---

### State subobjects

ほとんどのサブオブジェクトは API をミラーリングするが、例外はここに記載されている。

ランタイムは、DXIL ライブラリで直接定義されたサブジェクトを DDI でプレーンなサブオブジェクトとして渡します（ドライバは、取得した DXIL ライブラリでサブオブジェクトを見つけることはありません）。

また、ランタイムはデフォルトの関連付けを含むあらゆるサブオブジェクトの関連付けの定義を、エクスポートされた関数ごとに関連付けの明示的なリストに変換する。そのため、ドライバはどのサブオブジェクトがどの関数と関連付けられる必要があるかについての規則を理解しようとする必要はない。

---

#### D3D12DDI_STATE_SUBOBJECT_TYPE

```C++
typedef enum D3D12DDI_STATE_SUBOBJECT_TYPE
{
    D3D12DDI_STATE_SUBOBJECT_TYPE_STATE_OBJECT_CONFIG = 0, // D3D12DDI_STATE_OBJECT_CONFIG_0054
    D3D12DDI_STATE_SUBOBJECT_TYPE_GLOBAL_ROOT_SIGNATURE = 1, // D3D12DDI_GLOBAL_ROOT_SIGNATURE_0054
    D3D12DDI_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE = 2, // D3D12DDI_LOCAL_ROOT_SIGNATURE_0054
    D3D12DDI_STATE_SUBOBJECT_TYPE_NODE_MASK = 3, // D3D12_NODE_MASK_0054
    // 4 unused
    D3D12DDI_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY = 5, // D3D12DDI_DXIL_LIBRARY_DESC_0054
    D3D12DDI_STATE_SUBOBJECT_TYPE_EXISTING_COLLECTION = 6, // D3D12DDI_EXISTING_COLLECTION_DESC_0054
    // skip value from API not needed in DDI
    // skip value from API not needed in DDI
    D3D12DDI_STATE_SUBOBJECT_TYPE_RAYTRACING_SHADER_CONFIG = 9, // D3D12DDI_RAYTRACING_SHADER_CONFIG_0054
    D3D12DDI_STATE_SUBOBJECT_TYPE_RAYTRACING_PIPELINE_CONFIG = 10, // D3D12DDI_RAYTRACING_PIPELINE_CONFIG_0054
    D3D12DDI_STATE_SUBOBJECT_TYPE_HIT_GROUP = 11, //
    D3D12DDI_HIT_GROUP_DESC_0054

    // DDI only objects
    D3D12DDI_STATE_SUBOBJECT_TYPE_SHADER_EXPORT_SUMMARY = 0x100000, // D3D12DDI_FUNCTION_SUMMARY_0054
} D3D12DDI_STATE_SUBOBJECT_TYPE;
```

ドライバが見るステートオブジェクトタイプ。このリストは、API に存在するものと全く同じではないことに注意してください。d3d12_state_object_type。上で議論したように、API での関連サブオブジェクトは、関数ごとの関連リスト - D3D12DDI_FUNCTION_SUMMARY_0054 に変換されます。

他の様々な DDI サブオブジェクトの定義は、ここでドキュメント化された API の同等物と同一であるため、省略します。DDI サブオブジェクトの定義が若干異なるマイナーな例外： API でサブオブジェクトが API COM インターフェースを含む場合、DDI は DDI ハンドルを含みます。例えば、D3D12_GLOBAL_ROOT_SIGNATURE の ID3D12RootSignature\*は、 `D3D12DDI_GLOBAL_ROOT_SIGNATURE_0054` という DDI 同等のサブオブジェクトに現れ、 `D3D12DDI_HOOTSIGNATURE` の hGlobalRootSignature と一緒に現れます。

---

#### D3D12DDI_STATE_SUBOBJECT_0054

```C++
typedef struct D3D12DDI_STATE_SUBOBJECT_0054
{
    D3D12DDI_STATE_SUBOBJECT_TYPE Type;
    const void* pDesc;
} D3D12DDI_STATE_SUBOBJECT_0054;
```

ParameterDefinition`D3D12DDI_STATE_SUBOBJECT_TYPE Type`Type of subobject. See [D3D12DDI_STATE_SUBOBJECT_TYPE](#d3d12ddi_state_subobject_type).`const void* pDesc`<p>Pointer to subobject whose contents depend on Type. For instance if Type is `D3D12DDI_STATE_SUBOBJECT_TYPE_LOCAL_ROOT_SIGNATURE`, the pDesc points to `D3D12DDI_GLOBAL_ROOT_SIGNATURE_0054`.</p><p>Any data within the subobjects including shader bytecode is remains valid for the lifetime of the state object, as the memory is owned by the runtime, with no references to data the app passed in to CreateStateObject. So the app doesn't have to keep its creation data around and the driver doesn't need to make copies of any part of a state object definition.</p>---

#### D3D12DDI_STATE_SUBOBJECT_TYPE_SHADER_EXPORT_SUMMARY

D3D12DDI_STATE_SUBOBJECT_TYPE_SHADER_EXPORT_SUMMARY は、ランタイムのステートオブジェクト検証および作成プロセス中に生成される DDI サブオブジェクトです。これは、ランタイムが決定した情報、例えば、どのサブオブジェクトが任意のエクスポートに関連付けられたか、デフォルトの解決を含む情報を提供します。

サブオブジェクトの関連付けが API でどのように機能するかについての議論は、サブオブジェクトの関連付けの動作を参照してください。DDI では、ドライバは単に関連付けルールの結果を見るだけです。

---

#### D3D12DDI_FUNCTION_SUMMARY_0054

```C++
typedef struct D3D12DDI_FUNCTION_SUMMARY_0054
{
    UINT NumExportedFunctions;
    _In_reads_(NumExportedFunctions) const
    D3D12DDI_FUNCTION_SUMMARY_NODE_0054* pSummaries;
    D3D12DDI_EXPORT_SUMMARY_FLAGS OverallFlags;
} D3D12DDI_FUNCTION_SUMMARY_0054;
```

ステートオブジェクト内のすべてのエクスポートされた関数の説明。

ParameterDefinition`UINT NumExportedFunctions`How many functions are exported by the current state object. Size of pSummaries array.`_In_reads_(NumExportedFunctions) const D3D12DDI_FUNCTION_SUMMARY_NODE_0054* pSummaries`Array of [D3D12DDI_FUNCTION_SUMMARY_NODE_0054](#d3d12ddi_function_summary_node_0054).`D3D12DDDI_EXPORT_SUMMARY_FLAGS OverallFlags`Properties of the state object overall. See [D3D12_EXPORT_SUMMARY_FLAGS](#d3d12_export_summary_flags). This is the aggregate of the flags for each individual export. See Flags in [D3D12DDI_FUNCTION_SUMMARY_NODE_0054](#d3d12ddi_function_summary_node_0054).---

#### D3D12DDI_FUNCTION_SUMMARY_NODE_0054

```C++
typedef struct D3D12DDI_FUNCTION_SUMMARY_NODE_0054
{
    LPCWSTR ExportNameUnmangled;
    LPCWSTR ExportNameMangled;
    UINT NumAssociatedSubobjects;
    _In_reads_(NumAssociatedSubobjects) const
        D3D12DDI_STATE_SUBOBJECT_0054*const* ppAssociatedSubobjects;
    D3D12DDI_EXPORT_SUMMARY_FLAGS Flags;
} D3D12DDI_FUNCTION_SUMMARY_NODE_0054;
```

作成中のステート オブジェクトの各シェーダー エクスポート（関数のエクスポートではない）について、この構造体は、どのサブオブジェクトがそれに関連付けられたか、および未解決のバインディングが残っているなどの状態を示すいくつかのフラグ（これはコレクション ステート オブジェクトで有効）をドライバーに知らせます。

ParameterDefinition`LPCWSTR ExportNameUnmangled`Unmangled name of the export.`LPCWSTR ExportNameMangled`Mangled name of the export.`UINT NumAssociatedSubobjects`Number of subobjects associated with the export. Size of ppAssociatedSubobjects array.`_In_reads_(NumAssociatedSubobjects) const D3D12DDI_STATE_SUBOBJECT_0054*const* ppAssociatedSubobjects`Array of pointers to subobjects, [D3D12DDI_STATE_SUBOBJECT_0054](#d3d12ddi_state_subobject_0054), that are associated with the current export.`D3D12DDI_EXPORT_SUMMARY_FLAGS Flags`Status of the current export. See [D3D12_EXPORT_SUMMARY_FLAGS](#d3d12_export_summary_flags).---

#### D3D12_EXPORT_SUMMARY_FLAGS

```C++
typedef enum D3D12DDI_SHADER_EXPORT_SUMMARY_FLAGS
{
    D3D12DDI_SHADER_EXPORT_SUMMARY_FLAG_NONE = 0,
    D3D12DDI_SHADER_EXPORT_SUMMARY_FLAG_UNRESOLVED_RESOURCE_BINDINGS = 0x1,
    D3D12DDI_SHADER_EXPORT_SUMMARY_FLAG_UNRESOLVED_FUNCTIONS = 0x2,
    D3D12DDI_EXPORT_SUMMARY_FLAG_UNRESOLVED_ASSOCIATIONS = 0x4,
} D3D12DDI_SHADER_EXPORT_SUMMARY_FLAGS;
```

ランタイムが指定されたシェーダーエクスポートについて決定したプロパティ（呼び出す可能性のある関数のグラフを含む）を示すフラグです。実行可能なステートオブジェクト（RTPSO など）では、ランタイムはすべての依存関係が解決されるようにするので、未解決のリソースバインディングまたは未解決の関数は、コレクションステートオブジェクトにのみ表示されます。

フラグの設定に関係なく、ランタイムが完全なリンクを実行していないため、ドライバが DXIL ライブラリ間でコードをリンクしているときにコードの非互換性を見つける可能性が残っています。しかし、これはまれなケースです。例えば、ある DXIL ライブラリのシェーダーが、パラメータがローカルで定義されたユーザー定義型である関数を呼び出すことがあるとします。呼び出された関数は、別の DXIL ライブラリで同じ関数シグネチャで表示されるかもしれませんが、ユーザー定義型はそこで異なって定義されています。完全なリンクを行わない場合、ランタイムはこの点を見逃し、その場合、ドライバはステート・オブジェクトの作成に失敗しなければなりません。

---

#### State object lifetimes as seen by driver

##### Collection lifetimes

ステートオブジェクト（ `S` と呼ぶ）が、既存のコレクション（ `E` と呼ぶ）などの別のステートオブジェクトを参照する場合を考えてみましょう。）

ランタイムでは、 `S` は `E` の参照を保持しているので、アプリが `E` を破棄しても、 `S` が破棄されるまでドライバーは `E` の破棄を確認することができない。 これは、 `E` が最初に作成されたときに記述された DDI 構造体が、 `E` が生きている限り、生き続けることも意味します。 したがって、ドライバの中では、 `E` と `S` の両方が、 `E` の内容を記述したデータ構造をコピーすることなく自由に参照し続けることができるのです。

##### AddToStateObject parent lifetimes

AddToStateObject()によるインクリメンタルな追加の結果、親状態オブジェクト（ `P` と呼ぶ）の子である状態オブジェクト（ `C` と呼ぶ）を考えてみましょう。

ランタイムでは、 `C` は `P` の作成記述 DDI 構造体に関する参照のみを保持します。 したがって、ドライバでは、 `C` と `P` の両方が、 `P` の内容を記述するデータ構造への参照をコピーせずに自由に保持することができます。

しかし、ランタイムでは、 `C` は `P` の DDI オブジェクト/ハンドル上の参照を保持しません。 そのため、アプリが `P` を破棄すると、たとえ `C` がまだ生きていても、ドライバは `P` を破棄するように言われます。 これは、もしドライバが `C` がそれに依存しているため、 `P` に関連するドライバ内部データを生かしておく必要があるなら、適切な参照追跡を手動で行う必要があることを意味します。 その子孫である `C` が生きている間でも `P` の destroy をドライバに見せることによって、ドライバは `P` に関連する依存性のないデータを破壊する機会を得ます。

> In a typical [AddToStateObject()](#addtostateobject) scenario, and app might be making regular additions while discarding the (smaller) parents that are no longer needed. The driver gets to observe that this is happening while still being able to rely on DDI data structures for all the parts of the incrementally grown state object staying alive.

---

### Reporting raytracing support from the driver

---

#### D3D12DDI_RAYTRACING_TIER

```C++
typedef enum D3D12DDI_RAYTRACING_TIER
{
    D3D12DDI_RAYTRACING_TIER_NOT_SUPPORTED = 0,
    D3D12DDI_RAYTRACING_TIER_1_0 = 10,
} D3D12DDI_RAYTRACING_TIER;
```

レイトレーシングのサポートレベル。現在、all or nothing です。

これは、 `D3D12DDI_OPTIONS_DATA_0054` の RaytracingTier メンバです。

---

# Potential future features

これは、将来の潜在的な機能の非網羅的なリストです。このリストは、おそらく成長し進化していくでしょう。

---

## Traversal shaders

ボトムレベルのアクセラレーション構造は現在、三角形ジオメトリとプロシージャ ル ジオメトリ（AABB を使用）の 2 種類をサポートしています。

第 3 のタイプの Bottom-Level Acceleration Structure として、トラバーサルを追加することができます。このノードのゴールは、ノードに到達したレイを転送するために、他のアクセラレーション構造（もしあれば）を手続き的に決定することです。例えば、適切な LOD ジオメトリを含む別のアクセラレーション構造を選択して、レイを転送することにより、動的な LOD 選択を可能にします。

トラバーサルノードは、トラバーサルシェーダを見つけるためのシェーダテーブルへのインデックスを持つ AABB（手続き型ジオメトリのような）として定義されるでしょう。これらの AABB のいずれかがレイでヒットすると、参照されるトラバーサ ルシェーダが起動されます。  トラバーサルシェーダは非常にシンプルで、レイを選択した別のアクセラレーション構 造に「フォワード」する（またはレイをドロップする）だけかもしれません。

次の組込み関数は TraceRay() に似ていて、パラメータが少なくなっています。

void ForwardRay(RaytracingAccelerationStructure AccelerationStructure,

uint RayContributionToHitGroupIndex,

uint MultiplierForGeometryContributionToHitGroupIndex,

uint MissShaderIndex,

RayDesc レイ)。

転送されたレイのレイ処理は、いくつかの例外を除いて、TraceRay()と同じように動作します。  最も近いヒットシェーダは、元のレイと転送されたレイの間で最も低い T に対してのみ呼び出されます。  転送されたレイで AcceptHitAndEndSearch() が呼び出された場合、親レイでの検索も終了します(その後、通常通り最も近いヒットの選択が行われます)。

- Traversal Shader 自身がペイロードを検査できるようにしたいかもしれません (読み取り専用？)。

  - TraceRay() の他のパラメータ、例えばペイロード、レイフラグ、インスタンスカルマスクは、新しいレイと一緒に転送されます。

- レイは任意に定義することができ、親レイを変換したものになると思われます。

- ForwardRay の再帰には、ユーザが宣言した再帰の制限があり、ユーザが宣言した TraceRay の再帰の制限とは別かもしれません。

- そして/または、おそらく ForwardRay は、使用するボトムレベルのアクセラレーション構造を指定します（インスタンスデータの定義を含む）。

  - グラフィックスまたは計算パイプライン・ステートのシェーダー識別子をコマンドシグネチャに含めるオプション

---

## More efficient acceleration structure builds

現在の仕様では、アプリは加速構造構築前に GPU メモリに最終位置で頂点位 置を生成する必要があります（加速構造定義で利用可能なインスタンス変換 がたまたまその仕事をするのでない限り）。

変換された（たとえば、スキニングされた）頂点のメモリへのステージングを回避することができれば、より効率的である可能性があります。おそらく、進行中のアクセラレーション構造体の構築は、コンピュートシェーダ、 ストリーム出力、UAV の書き込み、および／またはスレッド協調頂点／プリミ ティブ処理専用のいくつかの新しいシェーダステージを通じて生成されるジ オメトリで直接供給することができます。おそらく、システムは、ラスタライゼーションの目的でシェーダが処理する際にジオメトリを直接取り込み、同時にそれを不透明で進行中のアクセラレーション構造構築にエンコードするのでしょう。このシステムは、アプリがジオメトリを処理する順序にすでに組み込まれている可能性のあるデータコヒーレンシーを利用することができ、GPU メモリへのステージングジオメトリの無駄な書き込み/読み込みを排除する可能性があります。

より高度なアクセラレーション構造の複雑さが、それに見合わない可能性もありま す。例えば、完成度のボトルネックが他の場所にある傾向がある場合です。

---

## Beam tracing

Ray 生成シェーダは、各スレッドのグリッド位置が空間の明示的に定義された領域 を表すという、異なる動作モードを持つことができます。この領域は、ビューポートの曲率のパラメータ化された定義における各「ピク セル」を通して目が見るであろうボリュームを表す錐台とすることができます。ビームは acceleration structure のジオメトリと交差し、交差したジオメトリのある種のラスタライズを生成し、おそらく深度バッファリングとマルチサンプリングが関与している可能性があります。シェーダの選択は、現在のレイトレーシングの提案と同様に、シェーダテーブルから来ることができます。システムは、ビューボリュームのパラメータ化とそれがどのようにビームに分割されるかを理解するので、従来のラスタライザが行うのと同様に、メモリレイアウトとラスタライズオーダーの最適化（「ピクセル」スパニング）を行うことができます。

このアプリケーションは、ジオメトリの投影とシェーディングの頻度をディスプレイ光学系と整合させるもので、従来のラスタライザで行われていたように、ピクセルの平面に希望する投影を押し込んでいたのとは対照的です。

---

## ExecuteIndirect improvements

---

### DispatchRays in command signature

これは現在サポートされています - CreateCommandSignature() を参照してください。

---

### Draw and Dispatch improvements

さらに進んで、おそらく、レイトレーシングでダイナミックなシェーダ選択を可能にするシェーダテーブルのコンセプトは、グラフィックスとコンピュートに戻って適用されるかもしれません。そうすれば、Draw*()/Dispatch*()を含むコマンドシグネチャでさえも、その両方が存在する可能性があります。

1. 使用するパイプライン・ステートを選択するシェーダーテーブルにインデックスを設定するオプション（必要であればローカルルート引数も含む

2. この一環として、コマンドシグネチャは、記述子テーブルの設定を可能にするよう改善されるべきで、ルートパラメータタイプのフルセットを変更することができ、これは記述子テーブルのサポートを追加することを意味します。

   - as part of this, command signatures should also be improved to
     allow descriptor table setting, so the full set of root
     parameter types can be changed which means adding descriptor
     table support

---

### BuildRaytracingAccelerationStructure in command signature

また、ExecuteIndirect()を使用して、acceleration structure のビルドを発行できるのも面白いかもしれません。これがどのように見えるかは、より効率的な acceleration structure 構築の設計がどのように行われるかに関係するかもしれません。

---

# Change log

VersionDateChange logv0.019/27/2017Initial draft.v0.0210/20/2017<li>Changed shader record alignment to 16 bytes</li><li>Changed argument alignment within shader records: each argument must be aligned to its size</li><li>Better naming for various parameters that contribute to shader indexing.</li><li>Ray recursion overflow or callable shader stack overflow now result in a to-be-defined exception mechanism being invoked (likely involving device removal).</li><li>Initial assumption was that wave intrinsics should be disallowed in raytracing shaders (preservingray scheduling independence). Switched to allowing them with the intent that PIX and careful apps can use them. The HLSL compiler may need to warn about how fragile this can be if misused in raytracing shaders.</li><li>Loosened the acceleration structure visualization mode for tools (PIX) to allow for tolerances and variation allowed within acceleration structure builds (to be fleshed out further elsewhere later).</li><li>Changed raytracing acceleration structure alignment from 16 to 256 bytes.</li><li>Clarified that raytracing shaders that can input ray payloads must declare their layout, even if they aren't looking at it.</li><li>Fleshed out discussion of acceleration structure determinism</li><li>Fleshed out raytracing API method documentation (state objects still to be done)</li><li>Added some detail to ray-triangle intersection specification (still more to be done)</li><li>Added some details to callable shader description</li><li>Added shader cycle counter spec to tools section, ported from the spec that is in D3D11 for shader model 5. What's missing is a way to correlate shader cycle counts to wall clock time.</li>v0.0311/02/2017<li>Specified that wave intrinsics have scope of validity bounded by thread repacking points in the shader (as well as start/end). Thread repacking points: CallShader(), TraceRay(), ReportIntersection().</li><li>Renames of some of the pieces in the state object API to be more consistent with DXIL library concepts.</li><li>Added a MissShaderIndex parameter to TraceRay(). So now the selection of miss shaders out of the miss shader table is no longer coupled to how the hit group is selected.</li><li>Added RAY*FLAG_SKIP_CLOSEST_HIT_SHADER. Also added an example in the ray flags section of the overall walkthrough, showing the usefulness of ray flags in general, including this new flag. Also documented each ray flag.</li><li>Added an instance culling section in the walkthrough (the feature was already present), highlighting the usefulness.</li><li>In the callable shaders discussion in the walkthrough, clarified that callable shaders are expected to be implemented by scheduling them for separate execution as opposed to being inlined. Discussed of how callable shaders are really no different than a special case of running a miss shader, minus any geometry interaction. So given an implementation has to support miss shaders, supporting callable shaders is in some ways a subset. And this should scope app expectations about how they behave accordingly.</li><li>Added a Watertightness requirement to the ray-triangle intersection spec (which is still a work in progress overall). Snippet: It is expected that implementations of watertight ray triangle intersections, including following a form of top-left rule to remove double hits on edges, are possible without having to resort to costly paths such as double precision fallbacks. A proposal is in the works and will appear in this spec.</li><li>Added D3D12_RESOURCE_STATE_SHADER_TABLE</li><li>Added bit size limits for the parameters to TraceRay that contribute to shader table indexing.</li><li>Added section describing potential future features, such as: traversal shaders, more efficient acceleration structure builds, beam tracing, ExecuteIndirect() improvements. This list will grow.</li><li>Clarified that if the raytracing process encounters a null shader identifier, no shader is executed for that purpose and the raytracing process continues.</li>v0.0411/07/2017<li>Called out the largest open issue with this spec as it stands now (mention of which was missing before). In short it has been observed that the current design may not be the best fit for a lot of current and near-future hardware given the way it steers towards ubershaders. While this isn't a unanimous opinion, it must be considered. This note appears both in the Open Issues section and repeated in the Design Goals section at the front -- look there for more detail.</li><li>Stopped allowing GeometryContributionToHitGroupIndex to change in bottom-level acceleration structure updates.</li><li>Removed RayRecursionLevel() given the overhead it might cause for implementations to be able to report it. If apps need it they can track it manually in ray payload.</li><li>Stated there isn't a limit on the user declared callable shader stack size. The question still remains of what the failure mode should be if too big a stack causes out of memory.</li><li>Clarified that acceleration structure serialization/deserialization (intended for PIX) isn't meant for caching acceleration structures. It is likely that doing a full build of an acceleration structure will be faster than loading it from disk.</li><li>Also stated that deserialize and visualize modes for acceleration structure copies are only allowed when the OS is in Developer Mode.</li><li>Minor cleanup of acceleration structure update flag behaviors: If an acceleration structure is built with ALLOW_UPDATE and then PERFORM_UPDATE is done, the result still allows update. ALLOW_UPDATE cannot be specified with PERFORM_UPDATE -- further updates are implied to be allowed for the result of an update.</li><li>For acceleration structure builds, added a D3D12_ELEMENTS_LAYOUT parameter describing how instances/geometries are laid out. There are two options: D3D12_ELEMENTS_LAYOUT_ARRAY and D3D12_ELEMENTS_LAYOUT_ARRAY_OF_POINTERS. This is similar to the flexibility on the array of rendertargets in OMSetRenderTargets in graphics.</li><li>For wave intrinsics in raytracing shaders added open issue that an explicit AllowWaveRepacking() intrinsic may be required to be paired with intrinsics like TraceRay() if an app wants to use other wave intrinsics in the shader.</li><li>HLSL syntax cleanup and minor renaming, such as:</li><li>Renamed PrimitiveID() to PrimitiveIndex() since it is more like InstanceIndex(), being autogenerated as opposed to being user defined, like InstanceID()</li><li>Started documenting the required resource state for resources in some of the APIs (not finished yet). Also listed which APIs can work on graphics vs compute vs bundle command lists.</li>v0.0511/15/2017<li>Clarified that on acceleration structure update, while the transform member of geometry descriptions can change, it can't change from NULL &lt;-&gt; non-NULL.</li><li>Clarified that for tools visualization of an acceleration structure, the returned geometry may have transforms folded in and may change topology/format of the data (with no loss of precision), so strips might become lists and float16 positions might become float32.</li><li>DispatchRays() can be called from graphics or compute command lists (had listed graphics only)</li><li>Consistent with above, raytracing shaders that use a root signature (instead of, or in addition to a local root signature) get root arguments from the compute root arguments on the command list.</li><li>Clarified in shader table memory initialization section that descriptor handles (identifying descriptor tables) take up 8 bytes.</li><li>For RayTracingAccelerationStructure clarified that the HLSL declaration includes :register(t#) binding assignment syntax (just like any other SRV binding). Also clarified that acceleration structures can be bound as root descriptors as well as via descriptor heap.</li><li>Switched the way descriptor based acceleration structure SRVs are declared. Instead of just being an acceleration structure flag on the buffer view type, switched to having a new view type: D3D12_SRV_DIMENSION_RAYTRACING_ACCELERATION_STRUCTURE. The description for this view type contains a GPUVA, which makes more sense than legacy buffer SRV creation fields: FirstElement, NumElements etc. -- these don't mean anything for acceleration structures.</li>v0.061/29/2018<li>Changed LPCSTR parameters in state objects (for exports names) to LPWSTR to be consistent with DXIL APIs</li><li>Minor enum renames</li><li>Made a few fields D3D12_GPU_VIRTUAL_ADDRESS_RANGE (instead of just \_ADDRESS) for APIs that are writing to memory, to help tools and drivers be simpler</li><li>Added a NO_DUPLICATE_ANYHIT_INVOCATION geometry flag (enabling faster acceleration structures if absent)</li><li>Removed GeometryContributionToHitGroupIndex field in D3D12_RAYTRACING_GEOMETRY_DESC in favor of it being a fixed function value: the 0 based index of the geometry in the order the app supplied it into the bottom- level acceleration structure. Having this value be user programmable turned was not hugely valuable, and there was IHV pushback against supporting it until a stronger need came up (or perhaps some better shader table indexing scheme is proposed that can be implemented just as efficiently).</li><li>Added struct defining acceleration structure serialization header</li><li>Removed topology and strip cut value from geometry desc. Just triangle lists now.</li><li>Clarified that during raytracing visualization, geometries may appear in a different order than in the original build, as this order has no bearing on raytracing behavior.</li><li>Specified that intersections shaders are allowed to execute redundantly for a given ray and procedural primitive (redundant, but implementation is free to make that choice). Detailed a bunch of caveats related to this.</li><li>Defined ray recursion limit to be declared by app in the range [0…31].<li><li>GetShaderIdentifierSizeInBytes() returns a nonzero multiple of 8 bytes.</li><li>Added RayFlags() intrinsic, returning current ray flags to all shaders that interact with rays.</li><li>Changed callable shader stack limit declaration to just a shader stack limit that applies to all raytracing shaders except ray generation shaders (since they are the root).</li><li>Minor corrections to shader example code.</li><li>Renamed InstanceCullMask to InstanceInclusionMask (and flipped the sense of the instance masking test). Was: InstanceCullMask &amp; InstanceMask -&gt; cull. Changed to !(InstanceInclusionMask &amp; InstanceMask) -&gt; cull. Feedback was the change made the feature more useful typically, despite the slightly awkward implication that if apps set InstanceMask to 0 in an instance definition it will never get included.</li><li>For the HitGroupTable parameter to DispatchRays, specified: all shader records in the table must contain a valid shader identifier (NULL is valid). In contrast, local root arguments in the shader records need only be initialized if they could be referenced during raytracing (same requirement for any type of shader table).</li><li>Required that GPU descriptor handles are globally unique across all descriptor heaps regardless of type. This makes it much cheaper for tools/debug layer to validate descriptor handles in shader tables, including making sure.</li><li>Before deserializing a serialized top-level acceleration structure the app patches the pointers in the header to where bottom-level acceleration structures will be. Those bottom-level acceleration structures pointed to need not have been deserialized yet (spec previously required it), as long as they are deserialized by the app before raytracing uses the acceleration structure.</li><li>Removed D3D12_RESOURCE_STATE_SHADER_TABLE in favor of simply reusing D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE.</li><li>Required that resources that will contain acceleration structures must be created in the D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE state and must stay in that state forever. Synchronization of read/write operations to acceleration structures is done by UAV barrier. See "Acceleration structure memory restrictions" for more details.</li><li>For all raytracing APIs that input or output to GPU memory, defined for each relevant parameter what resource state it needs to be in. In summary: all acceleration structure accesses (read or write) require ACCELERATION_STRUCTURE state as described above. Any other read only GPU input to a raytracing operation must be in NON_PIXEL_SHADER_RESOURCE state, and destination for writes (such as post build info or serialized acceleration structures) must be in UNORDERED_ACCESS.</li><li>Added D3D12_ROOT_SIGNATURE_FLAG_LOCAL_ROOT_SIGNATURE so that apps can distinguish whether a root signature is meant for graphics/compute vs being a local root signature. This distinction between the two types of root signatures is useful for drivers since their implementation of each layout could be different. Also, local root signatures don't have limits on the number of root parameters that root signatures do.</li><li>Related to adding the local root signature flag above, removed the D3D12_SHADER_VISIBILITY_RAYTRACING option for root parameter visibility. In a root signature (as opposed to local root signature), the root arguments are shared with compute, which just uses D3D12_SHADER_VISIBILITY_ALL.</li><li>Clarified in the D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE description that winding order applies in object space. So instance transforms with a negative determinant (e.g. mirroring an instance) don't change the winding within the instance. Per-geometry transforms, by contrast, are combined with geometry in object space, so negative determinant transforms do flip winding in that case.</li><li>In the DDI section, added discussion of a shader export summary that the runtime will generate for drivers, indicating for each shader export in a collection or RTPSO which subobjects have been associated with it. In the case of a collection it also indicates if the export has any unresolved bindings or functions. With all this information, a driver can quickly decide if it should even bother trying to compile the shader yet. There are more details than summarized here.</li><li>Specified that collections can't be made from existing collections (for simplicity). Only executable state objects (e.g. RTPSOs) can be constructed using existing collections as part of their definition.</li>v0.072/1/2018<li>Renames to go with marketing branding: "DirectX Raytracing":<ul><li>RAY_TRACING -&gt; RAYTRACING,</li><li>RayTracing -&gt; Raytracing,</li><li>ray tracing -&gt; raytracing etc.</li><li>Getting rid of the space is meant to at least reduce the likelihood of people concatenating the name to RT, a confusingly overloaded acronym (runtime, WindowsRT etc.).</li></ul></li><li>Renamed various shader intrinsics for clarity:<ul><li>TerminateRay() -&gt; AcceptHitAndEndSearch(),</li><li>IgnoreIntersection() -&gt; IgnoreHit(),</li><li>ReportIntersection() -&gt; ReportHit(),<li><li>RAY_FLAG_TERMINATE_ON_FIRST_HIT -&gt; RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH</li></ul></li><li>Added "Subobject association behavior" section discussing details about the subobject association feature in state objects. Lots of detail into default associations, explicit associations, overriding associations, what scopes are affected etc. This might need more refinement or clarification in the future but should be a good start.</li><li>The destination resource state for acceleration structure serialize is UNORDERED_ACCESS (spec had mistakenly stated NON_PIXEL_SHADER_RESOURCE</li><li>Defined that the order of serializing top-level and/or bottom-level acceleration structures doesn't matter (and the dependencies don't have to still be present/intact in memory). Same for deserializing.</li><li>Got rid of "TBD: GeometryIndex()" HLSL intrinsic, as this value is not available to shaders. GeometryContributionToHitGroupIndex is only used in shader table indexing, and if apps want the value in the shader, they need to put a constant in their root signature manually.</li><li>Removed stale comment indicating there's 4 bytes padding between geometry descs in raytracing visualization output to allow natural alignment when viewed on CPU. That padding became unnecessary as of spec v0.06 when the GeometryContributionToHitGroupIndex was removed from geometry desc struct.</li>v0.082/6/2018<li>Series of wording cleanups and clarifications throughout spec</li><li>Clarified that observable duplication of primitives is what is not allowed in acceleration structures. Observable as in something that becomes visible during raytracing operations beyond just performance differences. Listed exceptions where duplication can happen: intersection shader invocation counts, and when the app has not set the NO_DUPLICATE_ANYHIT_INVOCATION flag on a given geometry</li><li>Changed the behavior of pipeline stack configuration. Moved it out of the raytracing pipeline config subobject (which was a part of the pipeline state) to being mutable on a pipeline state via GetPipelineStackSize()/SetPipelineStackSize(). Added detailed discussion on the default stack size calculation (which is conservative and likely wasteful) as well as how apps can optionally choose to set it with a more optimal value based on knowing their specific calling pattern.</li><li>Clarified that shader identifiers globally identify shaders, so that even if it appears in a different state object (with the same associations like any root signatures) it will have the same identifier.</li><li>In addition to being in acceleration_structure state, resources that will hold acceleration structures must also have resource flag allow_unordered_access. This merely acknowledges that it is UAV barriers that are used for read/write synchronization. And that behind the scenes, implementations will be doing UAV style accesses to build acceleration structures (despite the API resource state being always "acceleration structure"</li>v0.093/12/2018<li>For a (tools) acceleration structure visualization, geometries must appear in the same order as in build (given that hit group table indexing is influenced by the order of each geometry in the acceleration structure, so the order needs to be preserved for visualization).</li><li>Vertex format DXGI_FORMAT_R16G16B16_FLOAT doesn't actually exist. Specified that DXGI_FORMAT_R16G16B16A16_FLOAT can be specified, and the A16 component gets ignored. Other data can be packed over the A16 part, such as setting the vertex stride to 6 bytes for instance.</li><li>For GetShaderStackSize(), required that to get the stack size for a shader within a hit group, the following syntax is used for the entrypoint string name: hitGroupNameshaderType, where hitGroupName is the name of the hit group export and shaderType is one of: intersection, closesthit, anyhit (all case sensitive). E.g. "myLeafHitGroupanyhit". Stack size is specific to individual shaders, but the overall hit group a shader belongs to can influence the stack size as well.</li><li>Guaranteed that shader identifier size (returned by GetShaderIdentifierSize() can be no larger than 64 bytes.</li><li>The Raytracing Emulation section has been updated with more concrete details on the Fallback Layer. It will be a stand-alone library that will be open-sourced publically that will closely mirror the DXR API. Developers will be free to pull the repro and tailor it for their own needs, or even send pull-requests with contributions.</li><li>Removed row_major modifier on matrix intrinsics definition (irrelevant).</li><li>Removed system value semantics, added language to clarify parameter requirements.</li><li>Added state object flags: D3D12_STATE_OBJECT_FLAG_ALLOW_EXTERNAL_DEPENDENCIES_ON_LOCAL_DEFINITIONS and D3D12_STATE_OBJECT_FLAG_ALLOW_LOCAL_DEPENDENCIES_ON_EXTERNAL_DEFINITIONS. Both of these apply to collection state objects only. In the absence of the first flag (default), drivers are not forced to keep uncompiled code around just because of the possibility that some other entity in a containing collection might call a function in the collection. In the absence of the second flag (default), contents of collections must be fully resolved locally (including necessary subobject associations) such that advanced implementations can fully compile code in the collection (not having to wait until RTPSO creation). These flags will be added to the OS post-March 2018/GDC, and in the meantime the default behavior (absence of the flags) is the only available current behavior.</li><li>Added discussion: Subobject associations for hit groups. Explains that subobject associations can be made to the component shaders in a hit group and/or to the hit group directly. Basically it is invalid if there are any conflicts -- see the text for full detail.</li>v0.917/27/2018<li>Cleared out references to experimental/prototype API and open issues</li><li>Local root signatures don't have the size constraint that global root signatures do (since local root signatures just live in shader tables in app memory).</li><li>All local root signatures referenced in an RTPSO that define static samplers must have matching definitions for any given s# that they define, and the total number of s# defined across global and local root signatures must fit within bind model limit.</li><li>Shader table start address must be 64 byte aligned</li><li>Shader record stride must be a multiple of 32 bytes</li><li>Noted that subobjects defined in DXIL did not make this first release.</li><li>Added table (Subobject association requirements) describing rules for each subobject type.</li><li>Drivers are responsible for implementing caching of state objects using existing services in D3D12.</li><li>Added discussions of "Inactive primitives and instances" as well as "Degenerate primitives and instances"</li><li>Fleshed out "Fixed function ray-triangle intersection specification" with a hypothetical top-left rule description. Implementations must do something like it to ensure no double hits or holes along shared edges.</li><li>Exceeding ray recursion limit or overflowing pipeline stack result in device removed.</li><li>Refined "Default pipeline stack size" calculation</li><li>Added "Determining raytracing support"</li><li>Cut "Shader Cycle Counter" feature, could bring it back in a future release.</li><li>Series of minor tweaks to API/DDI definitions to get to final release state, including fleshing out documentation for individual APIs/structs/enums.</li><li>Fleshed out descriptions of D3D12_STATE_OBJECT_FLAG_ALLOW_LOCAL_DEPENDENCIES_ON_EXTERNAL_DEFINITIONS and D3D12_STATE_OBJECT_FLAG_ALLOW_EXTERNAL_DEPENDENCIES_ON_LOCAL_DEFINITIONS. The absence of these flags in a state object configuration (which is the default) means drivers can compile code in collections without waiting until RTPSO create (aside from final linking on a clean implementation), and conversely don't have to keep extra stuff around in each collection on the chance that other parts of an RTPSO will depend on whats inside a given collection. Apps can opt in to cross-linkage basically.</li><li>D3D12_NODE_MASK state object definition was missing.</li><li>Added D3D12_HIT_GROUP_TYPE member to hit groups.</li><li>Removed GetShaderIdentifierSizeInBytes(). Shader identifers are now a fixed 32 bytes.</li><li>Refactored GetRaytracingAccelerationStructurePrebuildInfo() and BuildRaytracingAccelerationStructure() APIs to share the same input struct as a convenience. The latter API just has additional parameters to specify (such as actual buffer pointers).</li><li>Added the functionality of GetAccelerationStructurePostbuildInfo() as optional output parameters to BuildRaytracingAccelerationStructure(). So an app can request post build info directly as an additional output of a build (without extra barrier calls between doing the build and getting postbuild info), or it can still separately get postbuild info for an already built acceleration structure.</li><li>BUILD_FLAG_ALLOW_UPDATE can only be specified on an original build or an update where the source acceleration structure also specified ALLOW_UPDATE. Similarly, BUILD_FLAG_ALLOW_COMPACTION can only be specified on an initial acceleration structure build or on an update where the input acceleration structure also specified ALLOW_COMPACTION.</li><li>Renamed "D3D12_GPU_VIRTUAL_ADDRESS Transform" parameter to D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC to Transform3x4 to be more clear about the matrix layout. Similarly renamed "FLOAT Transform[12]" member of D3D12_RAYTRACING_INSTANCE_DESC to "FLOAT Transform[3][4]"</li><li>Added support for 16 bit SNORM vertex formats for acceleration structure build input.</li><li>Added buffer alignment (and or stride) requirements to various places in API where buffer locations are provided.</li><li>Added option to query the current size of an already built acceleration structure "POSTBUILD_INFO_CURRENT_SIZE". Useful for tools inspecting arbitrary acceleration structures.</li><li>Added D3D12_SERIALIZED_DATA_DRIVERS_MATCHING_IDENTIFIER to serialized acceleration structure headers. This is opaque versioning information the driver generates that allow an app to serialize acceleration structures to disk and later restore. On restore, the identifier can be compared with what the current device reports via new CheckDriverMatchingIdentifier() API to see if a serialized acceleration structure is compatible with the current device (and can be deserialized).</li><li>Added SetPipelineState1() command list API for setting RTPSOs (factored out of the parameters to DispatchRays(). Calling SetPipelineState1() unsets any pipeline state set by SetPipelineState() (graphics or compute). And vice versa.</li><li>Made the grid dimensions in DispatchRays 3D to be consistent with compute Dispatch(). Similarly in HLSL, DispatchRaysIndex() returns uint3 and DispatchRaysDimensions() returns uint3.</li><li>Clarified when it is valid to call GetShaderIdentifier() to get the identifier for a given export from a state object.</li><li>Replaced float3x4 ObjectToWorld() and WorldToObject() intrinsics with ObjectToWorld3x4(), ObjectToWorld4x3(), WorldToObject3x4() and WorldToObject4x3(). The 3x4 and 4x3 variants just let apps pick whichever transpose of the matrix they want to use.</li><li>Documented the various DDIs that are not just trivial passthroughs of APIs. Most interesting of which is D3D12DDI_FUNCTION_SUMMARY_0054, a DDI only subobject in state object creation that the runtime generates, listing for each export what subobject got associated to it among other things. This way drivers don't need to understand the various rules for how subobject associations (with defaults) work.</li>v1.010/1/2018<li>Clarified that ray direction does not get normalized.</li><li>Clarified that tracing rays with a NULL acceleration structure causes a miss shader to run.</li><li>Set maximum shader record stride to 4096 bytes.</li><li>Reduced maximum local root signature size from unlimited to max-stride (4096) -- shader identifier size (32) = 4064 bytes max.</li><li>Corrected stale reference to DispatchRays() taking a state object as input -&gt; corrected to SetPipelineState1 as the API where state objects are set.</li><li>The grid dimensions passed to DispatchRays() must satisfy width*height*depth &lt;= 2^30. Exceeding this produces undefined behavior.</li><li>Removed stale mention that collections can be nested (they cannot at least for now).</li><li>Added geometry limits section: Max geos in a BLAS: 2^24, Max prims in BLAS across geos: 2^29, max instance count in TLAS 2^24.</li><li>Added Ray extents section: TMin must be nonnegative and &lt;= TMax. +Inf is ok for either (really only makes sense for TMax). Ray-trinagle intersection can only occur if the intersection T-value satisfies TMin &lt; T &lt; TMax. No part of ray origin/direction/T range can be NaN.</li><li>Added General tips for building acceleration structures section</li><li>Added Choosing acceleration structure build flags section</li><li>Added preamble to Raytracing emulation section pointing out that the fallback is no longer being maintained given the cost/benefit is not justified etc.</li><li>Various minor typos fixed</li><li>Fleshed out details of ALLOW_COMPACTION, including that specifying the flag may increase acceleration structure size.</li><li>Also for postbuild info, clarified some guarantees about CompactedSizeInBytes, such as compacted acceleration structures don't use more space than non compacted ones. And there isn't a guarantee that an acceleration structure with fewer items than another one will compact to a smaller size if both are compacted (the property does hold if they are not compacted).</li><li>Per customer request, clarified for D3D12_RAYTRACING_INSTANCE_DESC that implementations transform rays as opposed to transforming all geometry/AABBs.</li><li>Only defined ray flags are propagated by the system, e.g. visible to the RayFlags() shader intrinsic.</li><li>Clarified that the scope of shared edges where watertight ray triangle intersection applies only within a given bottom level acceleration structure for geometries that share the same transform.</li>v1.0111/27/2018<li>Clarified that determination of inactive/degenerate primitives includes all geometry transforms. For instance, per-geometry bottom level transforms are one way that NaNs can be injected to cheaply cause groups of geometry to be deactivated in builds.</li><li>Some clarifications in "Conflicting subobject associations" section regarding DXIL defined subobjects (a feature for a future release).</li><li>Clarified that mixing of geometry types within a bottom-level acceleration structure is not supported.</li>v1.023/4/2019Added a comment on ways of calculating a hit position and their tradeoffs to Closest Hit Shader section.v1.033/25/2019<li>Ported DXR spec to Markdown.</li><li>Removed wording that DXIL subobjects aren't in initial release, as they have now been implemented as of 19H1.</li>v1.044/18/2019<li>Support for indirect DispatchRays() calls (ray generation shader invocation) via [ExecuteIndirect()](#executeindirect).</li><li>Support for [incremental additions to existing state objects](#incremental-additions-to-existing-state-objects) via [AddToStateObject()](#addtostateobject).</li><li>Support for [inline raytracing](#inline-raytracing) via [TraceRayInline()](#rayquery-tracerayinline).</li><li>Added [D3D12_RAYTRACING_TIER_2](#d3d12_raytracing_tier), which requires the above features.</li>v1.055/15/2019<li>Renamed `InlineRayQuery` to simply `RayQuery` in support of adding this object as a return value for [TraceRay()](#traceray) (in addition to [TraceRayInline()](#rayquery-tracerayinline)). For `TraceRay()`, the returned `RayQuery` object represents the final state of ray traversal after any shader invocations have completed. The caller can retrieve information about the final state of traversal (CLOSEST_HIT or MISS) and relevant metadata such as hit attributes, object ids etc. that the system already knows about without having to manually store this information in the ray payload.</li><li>Added stipulations on the state object definition passed into the [AddToStateObject()](#addtostateobject) API based around the incremental addition being a valid self contained definition of a state object, even though it is being combined with an existing one. The list of stipulations are intended to simplify driver compiler burden, runtime validation burden, as well as simplify code linkage/dependency semantics.<li>Added `D3D12_STATE_OBJECT_FLAG_ALLOW_STATE_OBJECT_ADDITIONS` [flag](#d3d12_state_object_flags) so that *executable* state objects (e.g. raytracing pipelines) must opt-in to being used with [AddToStateObject()](#addtostateobject). The [flags](#d3d12_state_object_flags) discussion includes a description of what it means for *collection* state objects to set this flag.</li><li>Renamed D3D12_RAYTRACING_TIER_2_0 to D3D12_RAYTRACING_TIER_1_1 to slightly better convey the relative significance of the new features to the set of functionality in TIER_1_0.</li><li>Removed RAY_QUERY_STATE_UNINITIALIZED from RAY_QUERY_STATE in favor of simply defining that accessing an uninitialized `RayQuery` object produces undefined behavior, and the HLSL compiler will attemp to disalllow it.</li>v1.065/16/2019<li>Removed a feature proposed in previous spec update: ability for [TraceRay()](#traceray) to return a [RayQuery](#rayquery) object. For some implementations this would actually be slower than the application manually stuffing only needed values into the ray payload, and would require extra compilation for paths that need the ray query versus those that do not. This feature could come back in a more refined form in the future. This could be allowing the user to declare entries in the ray payload as system generated values for all query data (e.g. SV_CurrentT). Some implementations could choose to compile shaders to automatically store these values in the ray payload (as declared), whereas other implementations would be free to retrieve these system values from elsewhere if they are readily available (avoiding payload bloat).</li>v1.076/5/2019<li>For [Tier 1.1](#d3d12_raytracing_tier), [GeometryIndex()](#geometryindex) intrinsic added to relevant raytracing shaders, for applications that wish to distinguish geometries manually in shaders in adddition to or instead of by burning shader table slots.</li><li>Revised [AddToStateObject()](#addtostateobject) API such that it returns a new ID3D12StateObject interface. This makes it explicitly in the apps control when to release the original state object (if ever). Clarified lots of others detail about the semantics of the operation, particularly now that an app can produce potentially a family of related state objects (though a straight line of inheritance may be the most likely in practice).</li><li>Refactored [inline raytracing](#inline-raytracing) state machine. Gave up up original proposal's conceptual simplicity (matching the shader based model closely) in favor of a model that's functionally equivalent but makes it easier for drivers to generate inline code. The shader has a slightly higher burden having to do a few more things manually (outside the system's tracking). The biggest win should be that there is one logical place where the bulk of system implemented traversal work needs to happen: [RayQuery::Proceed()](#rayquery-proceed), a method which the shader will typically use in a loop condition until traversal is done. Some helpful links:<ul><li>[Inline raytracing overview](#inline-raytracing)</li><li>[TraceRayInline control flow](#tracerayinline-control-flow)</li><li>[RayQuery object](#rayquery)<li>[RayQuery intrinsics](#rayquery-intrinsics)</li><li>[TraceRayInline examples](#tracerayinline-examples)</li></ul>v1.087/17/2019<li>Removed distinction between inline ray flags and ray flags. Both dynamic ray flags for [RayQuery::TraceRayInline()](#rayquery-tracerayinline) and [TraceRay()](#traceray) as well as the template parameter for [RayQuery](#rayquery) objects share the same set of flags.</li><li>Added [ray flags](#ray-flags): `RAY_FLAG_SKIP_TRIANGLES` and `RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES` to allow completely skipping either class of primitives. These flags are valid everywhere ray flags can be used, as summarized above.</li><li>Allowed [RayQuery::CommitProceduralPrimitiveHit()](#rayquery-commitproceduralprimitivehit) and [RayQuery::CommitNonOpaqueTriangleHit()](#rayquery-commitnonopaquetrianglehit) to be called zero or more times per candidate (spec was 0 or 1 times). If `CommitProceduralPrimitiveHit(t)` is called multiple times per candidate, it requires the shader to have ensure that each time the new t being committed is closest so far in [ray extents](#ray-extents) (would be the new TMax). If `CommitNonOpaqueTriangleHit()` is called multiple times per candidate, subsequent calls have no effect, as the hit has already been committed.</li><li>[RayQuery::CandidatePrimitiveIndex()](#rayquery-candidateprimitiveindex) was incorrectly listed as not supported for HIT_CANDIDATE_PROCEDURAL_PRIMITIVE in the [RayQuery intrinsics](#rayquery-intrinsics) tables.</li><li>New version of raytracing pipeline config subobject, [D3D12_RAYTRACING_PIPELINE_CONFIG1](#d3d12_raytracing_pipeline_config1), adding a flags field, [D3D12_RAYTRACING_PIPELINE_FLAGS](#d3d12_raytracing_pipeline_flags). The HLSL equivalent subobject is [RaytracingPipelineConfig1](#raytracing-pipeline-config1). The available flags, `D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_TRIANGLES` and `D3D12_RAYTRACING_PIPELINE_FLAG_SKIP_PROCEDURAL_PRIMITIVES` (minus `D3D12*`when defined in HLSL) behave like OR'ing the above equivalent RAY_FLAGS listed above into any [TraceRay()](#traceray) call in a raytracing pipeline. Implementations may be able to make some pipeline optimizations knowing that one of the primitive types can be skipped.</li><li>Cut RayQuery::Clone() instrinsic, as it doesn't appear to have any useful scenario.</li><li>[Ray extents](#ray-extents) section has been updated to define a different range test for triangles versus procedural primitives. For triangles, intersections occur in the (TMin...TMax) (exclusive) range, which is no change in behavior. For procedural primitives, the spec wasn't explicit on what the behavior should be, and now the chosen behavior is that intersections occur in \[TMin...TMax\] (inclusive) range. This allows apps the choice about how to handle exactly overlapping hits if they want a particular one to be reported.</li>v1.0911/4/2019<li>Added [Execution and memory ordering](#execution-and-memory-ordering) discussion defining that [TraceRay()](#traceray) and/or [CallShader()](#callshader) invocations finish execution when the caller resumes, and that for UAV writes to be visible memory barriers must be used..</li><li>Added [State object lifetimes as seen by driver](#state-object-lifetimes-as-seen-by-driver) section to clarify how driver sees the lifetimes of collection state objects as well as state objects in a chain of [AddToStateObject()](#addtostateobject) calls growing a state object while the app is also destroying older versions.</li><li>In [BuildRaytracingAccelerationStructure()](#buildraytracingaccelerationstructure) API, tightened the spec for the optional output postbuild info that the caller can request. The caller can pass an array of postbuild info descriptions that they want the driver to fill out. The change is to require that any given postbuild info type can only be requested at most once in the array.</li>v1.111/7/2019<li>Added accessor functions [RayQuery::CandidateInstanceContributionToHitGroupIndex](#rayquery-candidateinstancecontributiontohitgroupindex) and [RayQuery::CommittedInstanceContributionToHitGroupIndex](#rayquery-committedinstancecontributiontohitgroupindex) so that inline raytracing can see the`InstanceContributionToHitGroupIndex`field in instance data. This is done in the spirit of letting inline raytracing see everything that is in an acceleration structure. Of course in inline raytracing there are no shader tables, so there is no functional behavior from this value. Instead it allows the app to use this value as arbitrary user data. Or when sharing acceleration structures between dynamic-shader-based raytracing, to manually implementing in the inline raytracing case the effective result of the shader table indexing in the dynamic-shader-based raytracing case.</li><li>In [Wave Intrinsics](#wave-intrinsics) section, clarified that use of [RayQuery](#rayquery) objects for inline raytracing does not count as a repacking point.</li>v1.113/3/2020<li>For [Tier 1.1](#d3d12_raytracing_tier), additional vertex formats supported for acceleration structure build input as part of [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc).</li><li>Cleaned up spec wording - [D3D12_RAYTRACING_PIPELINE_CONFIG](#d3d12_raytracing_pipeline_config) and [D3D12_RAYTRACING_SHADER_CONFIG](#d3d12_raytracing_shader_config) have the same rules: If multiple shader configurations are present (such as one in each [collection](#collection-state-object) to enable independent driver compilation for each one) they must all match when combined into a raytracing pipeline.</li><li>For [collections](#collection-state-object), spelled out the list of conditions that must be met for a shader to be able to do compilation (as opposed to having to defer it).</li><li>Clarified that [ray recursion limit](#ray-recursion-limit) doesn't include callable shaders.</li><li>Clarified that in [D3D12_RAYTRACING_SHADER_CONFIG](#d3d12_raytracing_shader_config), the MaxPayloadSizeInBytes field doesn't include callable shader payloads. It's only for ray payloads.</li>v1.124/6/2020<li>For`D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE_VISUALIZATION_DECODE_FOR_TOOLS`, clarified that the result is self contained in the destination buffer.</li>v1.137/6/2020<li>For [D3D12_RAYTRACING_GEOMETRY_TRIANGLES_DESC](#d3d12_raytracing_geometry_triangles_desc), clarified that if an index buffer is present, this must be at least the maximum index value in the index buffer + 1.</li><li>Clarified that a null hit group counts as opaque geometry.</li>v1.141/12/2021<li>Clarified that [RayFlags()](#rayflags) does not reveal any flags that may have been added externally via [D3D12_RAYTRACING_PIPELINE_CONFIG1](#d3d12_raytracing_pipeline_config1).</li>v1.153/26/2021<li>Added [payload access qualifiers](#payload-access-qualifiers) section, introducing a way to annotate members of ray payloads to indicate which shader stages read and/or write individual members of the payload. This lets implementations optimize data flow in ray traversal. It is opt-in for apps starting with shader model 6.6, and if present in shaders appears as metadata that is ignored by existing drivers. For shader model 6.7 and higher these payload access qualifiers are required to be used by default (opt-out).</li>v1.167/29/2021<li>For [any hit shaders](#any-hit-shaders), clarified that for a given ray, the system can't execute multiple any hit shaders at the same time. As such shaders can modify the ray payload freely without worrying about conflicting with other shader invocations.</li>v1.1710/25/2021<li>In [Degenerate primitives and instances](#degenerate-primitives-and-instances), added the clarification: An exception to the rule that degenerates cannot be discarded with`ALLOW_UPDATE`specified is primitives that have repeated index value can always be discarded (even with`ALLOW_UPDATE` specified). There is no value in keeping them since index values cannot be changed.</li>v1.183/31/2022<li>In [Inactive primitives and instances](#inactive-primitives-and-instances), changed handling of NaN in triangles to: Triangles are considered "inactive" (but legal input to acceleration structure build) if the x component of any vertex is NaN. The "any" used to be "each". This reduces the amount of undefined behavior apps are exposed to.</li>
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:

[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
[]:
