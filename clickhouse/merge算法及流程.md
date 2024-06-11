## 为什么需要merge？
与传统关系性数据库在写入时，数据根据主键或者其他方式将数据有序的写入文件。clickhouse则是根据分区part先放到对应的分区内，一次就是一个part非常的高效，但是part多了就会出现以下的问题：

    1. 查询时需要读取每个 Part 以做进一步的数据筛选，Part 数量过多会拖慢查询速度，读取每个 Part 无论大小都会有一定的固定开销，例如打开文件开销等
    2. 相似的数据不能在物理上存放在一起，数据局部性降低，需要更多的随机IO以及冗余读取，进而导致查询性能下降

按照LSM Tree的思想就要把一个个小的part合并成一个并且遵循一定的规则：  
1. 保证合并后的 Part 依然是有序的，因为这样可以加快查询速度
2. 因为不断有新的 Part 插入，合并操作应该是周期进行的，通常作为后台任务执行

## 什么时候合并？
在每次插入数据后，ClickHouse 便会在后台调度执行 Merge 任务。 但需要注意的是，并非每次都会真正执行
Merge 动作，因为有时会找不到合适的 Part 组合来执行 Merge
### 选择需要进行 Merge 的 Part
ClickHouse 首先进行了一个限制，只能对相邻的 Part 进行 Merge，显著降低了 Merge 算法的决策复杂度。相
邻的 Part 是在相近的时间进行插入的，具有时间上的局部性，将时间相近的数据合并在一起也有理论上的依据。  
```
SelectPartsDecision MergeTreeDataMergerMutator::selectPartsToMerge(){
        // 省略 ...

    // 将所有的 Parts 按照 Partition 分组

    // 当设置了 TTL 时，会尝试找出已经过了 TTL 期限的 Part，并新建称为 TTL_DELETE / TTL_RECOMPRESS 特殊 Merge 类型，并跳过后续的常规 Merge 任务

    // 具体实现位于 SimpleMergeSelector 
     SimpleMergeSelector::Settings merge_settings;
     if (aggressive)
         merge_settings.base = 1;

     parts_to_merge = SimpleMergeSelector(merge_settings)
                         .select(parts_ranges, max_total_size_to_merge);    
}
```
SimpleMergeSelector中base代码
```
SimpleMergeSelector::PartsRange SimpleMergeSelector::select(
    const PartsRanges & parts_ranges,
    const size_t max_total_size_to_merge)
{
    Estimator estimator;

    /// Precompute logarithm of settings boundaries, because log function is quite expensive in terms of performance
    const double min_size_to_lower_base_log = log(1 + settings.min_size_to_lower_base);
    const double max_size_to_lower_base_log = log(1 + settings.max_size_to_lower_base);

    for (const auto & part_range : parts_ranges)
        selectWithinPartition(part_range, max_total_size_to_merge, estimator, settings, min_size_to_lower_base_log, max_size_to_lower_base_log);

    return estimator.getBest();
}
```
在SimpleMergeSelector中base值比较重要，下面是“大模型”给出的解释  
base(对数底数（简称base）是控制写放大的主要调节参数。底数越大，写放大越小，但平均而言数据分区数量会增多。)
```
ClickHouse 中的 MergeTree 表引擎使用的合并算法中，“base”参数是一个关键的调控变量，用于决定在合并数据分区过程中一次合并多少个分区。在SimpleMergeSelector策略中，base 参数可以被视为对数底数，它影响着合并策略的行为方式及合并过程中的写放大程度。

具体来说：

写放大优化：增大 base 值意味着每次合并更多数据分区，这样可以减少总的合并次数，从而降低写放大效应。这是因为合并更多的数据会导致单次写入的数据量更大，相对减少了额外写入的数据量与原始数据量的比例。

数据分区数量：随着 base 值的增加，虽然单次合并的数据量变大，但在任意给定时刻，由于较少的合并次数，数据分区的数量可能会增多。然而，通过合理调整 base 参数，可以在写放大和数据分区数量之间寻求一种平衡。

动态调整：在 ClickHouse 中，base 参数不是恒定不变的，而是可以根据实际的数据分区数量、大小以及分区的年代等因素进行动态调整。通过线性插值或其他策略，使得合并策略更加灵活适应不断变化的数据集。

合并策略复杂性：考虑到数据分区大小可能不均匀，base 参数可以被解释为参与合并的各分区总体大小与最大分区大小之间的比率，这意味着即使分区大小各异，系统仍然可以尝试以较为均衡的方式进行合并。
```

![alt text](image.png)



## 怎么合并
### 两种 Merge 算法: Horizontal 与 Vertical
Merge 过程本身是针对主键的，对于非主键字段，可以选择以下两种处理方式:  
Horizontal: 在主键进行 Merge 时，其他非主键字段也同时连带写入结果集，直接完成 Merge  
Vertical: 在主键进行 Merge 时，暂时不管非主键字段，而是记录了结果集中的数据来自于哪里，等后续分别对其他非主键字段逐一进行处理。Vertical 方式更符合向量化执行的思想，每次仅处理一列数据，CPU Cache 命中率相比 Horizontal 方式会提升，因此执行效率会更高。  

Vertical 分为以下两个阶段:  
对主键涉及列执行 Merge，并生成一个名为 rows_sources 数组，记录结果集中的每一行分别来自于哪个 Part，注意这里仅需要记录来自于哪个 Part，不需要记录在 Part 中的位置。
对非主键字段，分别对每个字段，将所有原始 Part 对应的列作为输入，按照 rows_sources 记录的 Part 顺序，写入到新的列

下面截取clickhouse v19.18代码，流程都是一样的只不过最新版本解耦了
```
MergeTreeData::MutableDataPartPtr MergeTreeDataMergerMutator::mergePartsToTemporaryPart(){
    ...
    NamesAndTypesList gathering_columns;//gathering_columns 为除了 merging_columns 的剩余列
    NamesAndTypesList merging_columns;// merging_columns 为 PK 及 Index 相关列
    Names gathering_column_names, merging_column_names;
    extractMergingAndGatheringColumns(
        all_columns, data.sorting_key_expr, data.skip_indices,
        data.merging_params, gathering_columns, gathering_column_names, merging_columns, merging_column_names);
    ...        
    if (merge_alg == MergeAlgorithm::Vertical)
    {
        Poco::File(new_part_tmp_path).createDirectories();
        rows_sources_file_path = new_part_tmp_path + "rows_sources";//rows_source文件 在merge算法中会写入
        rows_sources_uncompressed_write_buf = std::make_unique<WriteBufferFromFile>(rows_sources_file_path);
        rows_sources_write_buf = std::make_unique<CompressedWriteBuffer>(*rows_sources_uncompressed_write_buf);
    }
    else
    {
        merging_columns = all_columns;
        merging_column_names = all_column_names;
        gathering_columns.clear();
        gathering_column_names.clear();
    }  
    ...
    for (const auto & part : parts)
    {
        auto input = std::make_unique<MergeTreeSequentialBlockInputStream>(
            data, part, merging_column_names, read_with_direct_io, true);
        ...
        src_streams.emplace_back(stream);
    } 
    ...
    switch (data.merging_params.mode)
    {
        //选择merge 排序算法
        case MergeTreeData::MergingParams::Ordinary:
            merged_stream = std::make_unique<MergingSortedBlockInputStream>(
                src_streams, sort_description, merge_block_size, 0, rows_sources_write_buf.get(), true, blocks_are_granules_size);
            break;  
        ...
    }
    ...
    //上面不同的part merge之后在这里读出来然后写到新的part中
    Block block; 
    while (!is_cancelled() && (block = merged_stream->read()))
    {
        rows_written += block.rows();

        to.write(block);

        merge_entry->rows_written = merged_stream->getProfileInfo().rows;
        merge_entry->bytes_written_uncompressed = merged_stream->getProfileInfo().bytes;   
        ...
    }   
    ...
    /// Gather ordinary columns
    if (merge_alg == MergeAlgorithm::Vertical)
    {
        ...
        //根据上面写入到的rows_source文件将普通列写入到对应的合并列中
        CompressedReadBufferFromFile rows_sources_read_buf(rows_sources_file_path, 0, 0);
        IMergedBlockOutputStream::WrittenOffsetColumns written_offset_columns;

        for (size_t column_num = 0, gathering_column_names_size = gathering_column_names.size();
            column_num < gathering_column_names_size;
            ++column_num, ++it_name_and_type)
        { 
            rows_sources_read_buf.seek(0, 0);
            ColumnGathererStream column_gathered_stream(column_name, column_part_streams, rows_sources_read_buf);

            MergedColumnOnlyOutputStream column_to(
                data,
                column_gathered_stream.getHeader(),
                new_part_tmp_path,
                false,
                compression_codec,
                false,
                /// we don't need to recalc indices here
                /// because all of them were already recalculated and written
                /// as key part of vertical merge
                std::vector<MergeTreeIndexPtr>{},
                written_offset_columns,
                to.getIndexGranularity());

            size_t column_elems_written = 0;

            column_to.writePrefix();
            //根据写入rows_source中的part信息读取其他列column_gathered_stream
            while (!merges_blocker.isCancelled() && (block = column_gathered_stream.read()))
            {
                column_elems_written += block.rows();
                column_to.write(block);
            }
            ...
            if (rows_written != column_elems_written)
            {
                throw Exception("Written " + toString(column_elems_written) + " elements of column " + column_name +
                                ", but " + toString(rows_written) + " rows of PK columns", ErrorCodes::LOGICAL_ERROR);
            }            
            ...                                 
        }      
        Poco::File(rows_sources_file_path).remove();   
    }    
}

```

总结一下：  
1. 默认选择 Horizontal 算法，当满足一定条件(体量大)时选择 Vertical 算法，因为 Vertical 有 overhead，需要落地一个文件  
2. Horizontal 与 Vertical 算法的不同在于对 gathering_columns 的处理，Vertical 对 gathering_columns 做了写入性能优化，而 Horizontal 没有这样的优化，逻辑更为简单  
3. 类 MergingSortedTransform 承担 Merge 具体的实现
![alt text](image-1.png)