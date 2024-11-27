
#### 前言


NET程序员是很幸福的,MS在上个月发布了`NET9.0`RTM,带来了不少的新特性,但是呢,还有很多同学软硬件都还没跟上时代的步伐,比如,自己的电脑还在跑Win7,公司服务器还在跑MSSQL2005\-2008的!


这不就引入了我们本文要探索的问题,因为MS早在`EFcore3.1`后就不再内置支持`ROW_NUMBER()`了,以至于需要兼容分页的代码都需要自行处理,当然同学们如果对EFCore没有依赖度也可以使用其他的ORM选型,当然如果不想折腾EFCore也能使用万能的RawSql拼接执行也是可以的 😃


最近自己发的Nuget包有个国外的程序员朋友提了一个Issue,以至于我马上行动起来


![image](https://img2024.cnblogs.com/blog/127598/202411/127598-20241126144141765-1932887759.png)


在`EFCore9`中, 以前兼容的好好的`ROW_NUMBER()`代码,升级尝鲜后发现跑不起来了,这主要是因为新版本的EFCore9做了很多破坏性更新,以至于我们不得不研究新的底层代码!


#### 兼容实现


之前发布过一个[Nuget包](https://github.com "Nuget包"),代码主要是基于以前道友兼容`EFCore7`适配到`EFCore8`的兼容,代码也不多变化也不大,不过呢,升级到`EFCore9`后发现底层的API全变了,不得不重新再实现一遍!


以下是兼容EFCore9的代码部分:



```
#if NET9_0_OR_GREATER

#pragma warning disable EF1001 // Internal EF Core API usage.

namespace Biwen.EFCore.UseRowNumberForPaging;

using Microsoft.EntityFrameworkCore.Query;
using System.Collections.Generic;

public class SqlServer2008QueryTranslationPostprocessorFactory(
    QueryTranslationPostprocessorDependencies dependencies,
    RelationalQueryTranslationPostprocessorDependencies relationalDependencies) : IQueryTranslationPostprocessorFactory
{
    private readonly QueryTranslationPostprocessorDependencies _dependencies = dependencies;
    private readonly RelationalQueryTranslationPostprocessorDependencies _relationalDependencies = relationalDependencies;

    public virtual QueryTranslationPostprocessor Create(QueryCompilationContext queryCompilationContext)
        => new SqlServer2008QueryTranslationPostprocessor(
            _dependencies,
            _relationalDependencies,
            queryCompilationContext);

    public class SqlServer2008QueryTranslationPostprocessor(QueryTranslationPostprocessorDependencies dependencies, RelationalQueryTranslationPostprocessorDependencies relationalDependencies, QueryCompilationContext queryCompilationContext) :
        RelationalQueryTranslationPostprocessor(dependencies, relationalDependencies, (RelationalQueryCompilationContext)queryCompilationContext)
    {
        public override Expression Process(Expression query)
        {
            query = base.Process(query);
            query = new Offset2RowNumberConvertVisitor(query, RelationalDependencies.SqlExpressionFactory).Visit(query);
            return query;
        }
        internal class Offset2RowNumberConvertVisitor(
            Expression root,
            ISqlExpressionFactory sqlExpressionFactory) : ExpressionVisitor
        {
            private readonly Expression root = root;
            private readonly ISqlExpressionFactory sqlExpressionFactory = sqlExpressionFactory;
            private const string SubTableName = "subTbl";
            private const string RowColumnName = "_Row_";//下标避免数据表存在字段

            protected override Expression VisitExtension(Expression node) => node switch
            {
                ShapedQueryExpression shapedQueryExpression => shapedQueryExpression.Update(Visit(shapedQueryExpression.QueryExpression), shapedQueryExpression.ShaperExpression),
                SelectExpression se => VisitSelect(se),
                _ => base.VisitExtension(node)
            };

            private SelectExpression VisitSelect(SelectExpression selectExpression)
            {
                var oldOffset = selectExpression.Offset;
                if (oldOffset == null)
                    return selectExpression;

                var oldLimit = selectExpression.Limit;
                var oldOrderings = selectExpression.Orderings;
                var newOrderings = oldOrderings switch
                {
                    { Count: > 0 } when oldLimit != null || selectExpression == root => oldOrderings.ToList(),
                    _ => []
                };

                var rowOrderings = oldOrderings.Any() switch
                {
                    true => oldOrderings,
                    false => [new OrderingExpression(new SqlFragmentExpression("(SELECT 1)"), true)]
                };

                var oldSelect = selectExpression;

                var rowNumberExpression = new RowNumberExpression([], rowOrderings, oldOffset.TypeMapping);
                // 创建子查询
                IList projections = [new ProjectionExpression(rowNumberExpression, RowColumnName),];

                var subquery = new SelectExpression(
                    SubTableName,
                    oldSelect.Tables,
                    oldSelect.Predicate,
                    oldSelect.GroupBy,
                    oldSelect.Having,
                    [.. oldSelect.Projection, .. projections],
                    oldSelect.IsDistinct,
                    [],//排序已经在rowNumber中了
                    null,
                    null,
                    null,
                    null);

                //构造新的条件:
                var and1 = sqlExpressionFactory.GreaterThan(
                    new ColumnExpression(RowColumnName, SubTableName, typeof(int), null, true),
                    oldOffset);
                var and2 = sqlExpressionFactory.LessThanOrEqual(
                    new ColumnExpression(RowColumnName, SubTableName, typeof(int), null, true),
                    sqlExpressionFactory.Add(oldOffset, oldLimit));

                var newPredicate = sqlExpressionFactory.AndAlso(and1, and2);

                //新的Projection:
                var newProjections = oldSelect.Projection.Select(e =>
                {
                    if (e is { Expression: ColumnExpression col })
                    {
                        var newCol = new ColumnExpression(col.Name, SubTableName, col.Type, col.TypeMapping, col.IsNullable);
                        return new ProjectionExpression(newCol, e.Alias);
                    }
                    return e;
                });


                // 创建新的 SelectExpression，将子查询作为来源
                var newSelect = new SelectExpression(
                    oldSelect.Alias,
                    [subquery],
                    newPredicate,
                    oldSelect.GroupBy,
                    oldSelect.Having,
                    [.. newProjections],
                    oldSelect.IsDistinct,
                    [],
                    null,
                    null,
                    null,
                    null);

                // projectionMapping replace
                var pm = new ProjectionMember();
                var projectionMapping = new Dictionary
                {
                    {
                        pm,
                        oldSelect.GetProjection(new ProjectionBindingExpression(null,pm,null))
                    }
                };
                newSelect.ReplaceProjection(projectionMapping);

                return newSelect;
            }
        }
    }
}

#pragma warning restore EF1001 // Internal EF Core API usage.

#endif

```

#### 最后


实现上逻辑还是一致的,反正都是将`Offset`转换为`ROW_NUMBER()`子查询中,取行号数据


只是代码实现区别有一些,以前的EFCore底层代码很多已经不在可用比如直接使用`PushdownIntoSubquery()`会报错,`GenerateOuterColumn()`内部的扩展方法发生了破坏性更新导致不能再使用等!


如果你的程序需要升级到`NET9`并还在使用早期数据库的话,可以引用我实现的代码部分,或者直接引用我发布的Nuget包



```
<PackageReference Include="Biwen.EFCore.UseRowNumberForPaging" Version="2.1.1" />

```

代码我放在了GitHub,任何问题欢迎Issue [https://github.com/vipwan/Biwen.EFCore.UseRowNumberForPaging](https://github.com):[wgetcloud加速器官网下载](https://longdu.org)


