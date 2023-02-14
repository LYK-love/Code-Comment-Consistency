# 1. Introduction

程序员在开发过程中经常会从其他项目中copy代码并进行重构，理想情况是在修改代码的同时，程序员会同步地修改注释。 然而也存在着代码或注释只有一方被修改的情况， 此时便发生了注释-代码的不一致。 这使得copy-refactor的代码块是“脆弱的”。由于程序员很大程度上依靠注释来理解代码和代码的不同部分之间的关系，以及它的用法和彼此之间的交流， 注释-代码的不一致性会对软件开发造成很大困扰， 因此有必要检查这种不一致性。 

为了方便分析，我们规定“程序单元”是一个代码块和其对应的注释， 给定程序单元$A$,  它被copy-refoactor为新程序单元$B$， 假定$A$是注释-代码一致的，我们将$B$与$A$做比较，判定$B$的注释-代码一致性。 

由于程序单元由代码块和注释两部分组成，我们的分析也分为两部分： (1) 分析$B$和$A$的代码是否一致。  (2) 分析$B$和$A$的注释是否一致。当且仅当$A$, $B$在(1)(2)中均一致，我们才认为$B$的注释-代码具有一致性。  注意到，如果$A$, $B$在(1)(2)中均不相同，我们就认为这是两个不相关的不同程序单元，因此也无所谓注释-代码的一致性， 毕竟该一致性只对在不同修改点的同一程序单元存在。

对于(1) ， 我们考虑最简单的情况：代码在文本层面上是否一致。 

对于(2)， 我们从语义层面分析注释的一致性。  由于注释的编写缺乏统一的规范，开发人员在编写注释时是非常自由的，注释可以用来描述不同种类的代码块，比如类、方法和语句； 注释也可以用来描述不同的内容，如总结功能，解释设计原理和描述实现细节，此外，由于注释是由自然语言编写的，它们本质上是模糊的，需要准确的语言学分析来获得注释的确切含义和范围。

例如，考虑Javadoc片段

```java
@param chrono Chronology to use, null means default
  in a school sealed envelope with the school seal or signature on the back side of the envelope.
```

它描述了"一个方法参数Chrono是一个Chronology类型的对象。null意味着默认". 这部分很难理解；它可能指定以某种 "默认 "方式处理null（例如，抛出一个NullPointerException），或者用null来代表Chronology的某种默认值。

为了更有效地理解注释，首先需要对注释进行**分类**。如前所述，注释是一个过于宽泛而难以统一处理的集合，自由的格式和内容让静态分类处理变得很困难， 单纯的基于NLP的方法过于泛化，对于“注释分类”这一特定领域并不能获得很高的精度。

 同时，注释作为一种特定的自然语言结构，对其分类可以做适当的简化。我们注意到，在半结构化注释的情况下， 注释天然地具有语义上比较稳定的结构，对此进一步分类也比较容易。 Java Doc是流行的半结构化注释，因此我们使用Java Doc注释作为样本进行注释分类的构建。在注释分类领域已经有了一些成熟的工作，徐翔哲等人的*CPC: Automatically Classifying and Propagating Natural Language Comments via Program Analysis*通过在注释分类中引入程序分析技术，取得了很显著的效果，我们就采用CPC作为注释分类工具， 将注释分成5类（CPC中构建的分类法，把注释分为5类），并以语句作为注释的基本单元。 对于$A$和$B$中的相同类别的注释，我们使用NLP技术来判定其语义一致性，如果于$A$和$B$中所有类别的注释都语义上一致，则认为于$A$和$B$的注释是一致的。注意到一个程序单元的一个类别的注释依然可能有许多条语句，此时如果两边的注释语句数量不相符，依然算作语义上不一致。

我们的贡献如下：

- 我们提出了在copy-refactor过程中的注释-代码一致性问题，并给予了规范的描述
- 我们提出了在copy-refactor过程中的注释-代码一致性检查，并给出了具体的思路和实现
- 我们提出了在检查copy-refactor过程中的注释-代码一致性时，由于注释自身的特性，有必要对注释进行分类，并采用CPC作为分类方法

# 2. Motivation

现代软件系统开发中， copy-refactor现象频繁地发生，然而据我们所知，现有的工作几乎没有以程序单元为单位，研究copy-refactor过程中的注释-代码不一致性问题的。 这种不一致性在软件系统中很普遍，有时甚至会造成严重的后果。我们用一个案例来说明copy-refactor时发生的注释-代码不一致性。

例子一如下:

```java
/**
 * @param loadedClass: 范型，被加载类
 * @return 布尔值，表明该类是否为抽象类
 */
private static boolean isAbstractModule(Class<?>
     loadedClass) 
{
	...
}
```

上例中，我们给出了`isAbstractModule（Class cls）`方法，该方法用于测试参数（类类型）是否是一个抽象类。 当开发者copy这份代码到自己的项目后，可能会：

**(1)** 修改方法，加入额外的逻辑，比如，如果参数是抽象类，则向标准输出输出一段内容。 

**(2)** 修改注释，如: (a) 再添加或删除一段注释，比如添加说明该方法内部逻辑的注释。 (b) 对已有的`@param`、 `@return`等Java Doc注释的内容做删改。

**(3)** 同时修改了代码和注释，比如新增加了一个方法参数，并同步更新了注释。

对于(3)， 尽管是对同一个程序单元的修改，但分析它的注释-代码一致性已经没有意义了，因为代码和注释均修改后的程序单元已经被视作一个新程序单元。**对这样的样本，我们不需要进一步分析其注释-代码一致性**。也就是说，我们只考虑(1)(2)， 即单独修改注释/代码的情况。



目前只考虑Java语言

# 3. Algorithm

## 3.1 Input

程序单元$A$, $B$

* 代码块$A$, $B$的代码部分为$\mathrm{Code}(A)$, $\mathrm{Code}(B)$
* 代码块$A$, $B$的注释部分为$\mathrm{Comm}(A)$,  $\mathrm{Comm}(B)$



注释分类：

* $\mathrm{Comm}(X)$可分类为$\mathrm{Type_A}(\mathrm{Comm}(X))$
  * 其中下标$A$代表注释分类$A$, 假设有5类注释，则对应下标$A$, $B$, $C$, $D$, $E$

## 3.2 Output

三元值， True和False表明$A$和$B$的代码-注释一致情况， $\mathrm{None}$表明$A$和$B$是两个不相干的程序单元，对其进行注释-代码一致性分析没有意义

* False iff 

$$
( \quad \mathrm{Code}(A) \ne \mathrm{Code}(B) \quad  \&\& \quad \mathrm{Comm}(A) = \mathrm{Comm}(B) \quad ) \quad || \quad \\
( \quad \mathrm{Comm}(A) \ne \mathrm{Comm}(B) \quad \&\&\mathrm{Code}(A) = \mathrm{Code}(B) \quad )
$$

* True iff $\mathrm{Code}(A) = \mathrm{Code}(B) \quad  \&\& \quad \mathrm{Comm}(A) = \mathrm{Comm}(B)$
* None iff $\mathrm{Code}(A) \ne \mathrm{Code}(B) \quad  \&\& \quad \mathrm{Comm}(A) \ne \mathrm{Comm}(B)$

## 3.3 Comment-Code Consistency Judgemtnt

* 注释-代码一致性判定算法:

  ```python
  def judge_Comment_Code_Consistentency( program_unit_A, program_unit_B ):
      isCodeIdentical = False
      isCommentIdentical = False
      
      # 程序单元A，B的注释
      comments_from_A = getComments(program_unit_A)
      comments_from_B = getComments(program_unit_B)
  
      code_block_from_A = getCodeBlock(program_unit_A)
      code_block_from_B = getCodeBlock(program_unit_B)
  		# 注释A经过CPC分类，分为五类注释，每一类注释都是一组注释单元
      typeA_comment_units_of_A, typeB_comment_units_of_A, typeC_comment_units_of_A, typeD_comment_units_of_A, typeE_comment_units_of_A = CPC(comments_from_A)
      
      typeA_comment_units_of_B, typeB_comment_units_of_B, typeC_comment_units_of_B, typeD_comment_units_of_B, typeE_comment_units_of_B = CPC(comments_from_B)
      
      # 先判断代码一致性
      isCodeIdentical = judgeIfCodeIdentical(code_block_from_A,code_block_from_B)
      
      # 再判断注释一致性
      # 对每一类注释进行比较
      is_typeA_CommentIdentical = judgeIfCommentIdentical(typeA_comment_units_of_A,typeA_comment_units_of_B)
      is_typeB_CommentIdentical = judgeIfCommentIdentical(typeB_comment_units_of_A,typeB_comment_units_of_B)
      is_typeC_CommentIdentical = judgeIfCommentIdentical(typeC_comment_units_of_A,typeC_comment_units_of_B)
      is_typeD_CommentIdentical = judgeIfCommentIdentical(typeD_comment_units_of_A,typeD_comment_units_of_B)
      is_typeE_CommentIdentical = judgeIfCommentIdentical(typeE_comment_units_of_A,typeE_comment_units_of_B)
  
      # 当且仅当所有类对注释都一致，才判定注释一致
      isCommentIdentical = is_typeA_CommentIdentical and is_typeB_CommentIdentical and is_typeC_CommentIdentical and is_typeD_CommentIdentical and is_typeE_CommentIdentical
  
      # 注释-代码一致
      if isCommentIdentical and isCodeIdentical:
          return True
  
      # 注释和代码只有一方一致，这就意味着注释-代码不一致
      elif ( isCommentIdentical or isCodeIdentical ) is True:
          return False
  
      # 注释-代码均不一致，这意味着两个参数是不同的程序单元，对其进行一致性分析没有意义
      else:
          return None
  ```

  

* 对于代码一致性，我们采用文本相似度计算工具，计算其文本相似度，原则上，我们认为只有两段代码在文本上完全相似，才认为两份程序单元的代码是一致的

  ```python
  # 程序单元A，B的代码块
  def judgeCodeIdentical( code_block_from_A, code_block_from_B ):
      return code_block_from_A == code_block_from_B
  ```

  

* 对于注释一致性，我们将注释采用之前介绍的CPC进行分类，得到总共属于5类的若干个注释单元（每个注释单元就是一条注释）， 将每个注释

  ```python
  def judgeIfCommentIdentical(comment_units_from_A, comment_units_from_B):
      isCommentIdentical = True
      for unit1 in comment_units_from_A:
          for unit2 in comment_units_from_B:
              # 使用NLP工具判断两个注释单元的相似度，这里以word2vec为例
              isSementicallyIdentical = word2vec(unit1, unit2)
              if isSementicallyIdentical is True:
                  continue
              else:
                  isCommentIdentical = False
                  break
          if isCommentIdentical is True:
              continue
          else:
              break
  
      return isCommentIdentical
  ```

# 4. Research Design



## 4.1 hypothesis

在注释-代码一致性分析方面，已经有一些现成的工具，我们的假设是，我们的方法效果比已有的工具好。



## 4.2 Research questions

1. 我们的方法在注释-代码一致性分析方面效果如何?
2. 使用注释分类是否能提升我们方法在注释-代码分析一致性时的有效性?
3. 我们的方法在开发者方面的帮助如何?



## 4.3 Method

### 4.3.1 RQ1 Efectiveness in Comment-Code Inconsistency Detection

为了回答RQ 1， 我们可以进行一项对比研究。我们将之前讲到的六个大型软件项目以程序单元为单位做分割，以分割后的程序单元作为样本， 分别输入其他工具和我们的工具。 并且用使用人工检验，雇佣多个实验者对样本的注释-代码有效性做检查，将人工检查的结果作为理想指标， 而工具的输出与理想指标的逼近程度，就是该工具的有效程度。

例如，假设对于100个程序单元，人工检查得到结果为：注释-代码不一致的程序单元有80个； 我们的工具结果为60个； 已有工具A的结果为40个， 则60比40更接近理想指标（80），则证明我们的工具效果更好，假设成立。

### 4.3.2 RQ2 Effectiveness in Importing Comment-Classfying Method

为了回答RQ2, 我们额外增加了原方法的不进行注释分类的版本，将该版本和原版本一起代入为回答RQ1所做的研究，原版本作为实验组，进行了注释分类； 新版本作为对照组，没有进行注释分类。 两个版本与人工检验得到的理想指标做比较，若原版本的效果比新版本好，就可以认为引入注释分类是可以提升我们方法的效果的。

新版本算法：

```python
# 注释一致性判定方法的不采用注释分类的版本，作为实验组，与采用了注释分类的原方法做对比。
def judge_Comment_Code_Consistentency_without_CPC(program_unit_A, program_unit_B):
    isCodeIdentical = False
    isCommentIdentical = False

    # 程序单元A，B的注释
    comments_from_A = getComments(program_unit_A)
    comments_from_B = getComments(program_unit_B)

    code_block_from_A = getCodeBlock(program_unit_A)
    code_block_from_B = getCodeBlock(program_unit_B)

    # 注释A经过CPC分类，分为五类注释，每一类注释都是一组注释单元
    comment_units_from_A = get_comment_units_without_CPC(comments_from_A)
    comment_units_from_B = get_comment_units_without_CPC(comments_from_B)

    # 先判断代码一致性
    isCodeIdentical = judgeIfCodeIdentical(code_block_from_A, code_block_from_B)

    # 再判断注释一致性
    # 没有注释分类的过程，因此只需判断所有注释单元的一致性
    isCommentIdentical = judgeIfCommentIdentical(comment_units_from_A, comment_units_from_B)

    # 注释-代码一致
    if isCommentIdentical and isCodeIdentical:
        return True

    # 注释和代码只有一方一致，这就意味着注释-代码不一致
    elif (isCommentIdentical or isCodeIdentical) is True:
        return False

    # 注释-代码均不一致，这意味着两个参数是不同的程序单元，对其进行一致性分析没有意义
    else:
        return None
```



### 4.3.3 RQ3 Usefulness in Helping Developers

回答RQ1的研究同样可以回答RQ3， 只要我们的方法的准确性与人工分类结果（即理想指标）足够接近，就说明我们的方法的效果足够好，与用户的实际理解足够相似.





## 4.4 Methodological characteristics

| characteristic         |                                               |
| ---------------------- | --------------------------------------------- |
| hypothesis             | 我们提出的工具的效果比已有的工具好            |
| type of experiment     | two group design                              |
| experimental unit      | Interviews                                    |
| experimental treatment | 将程序单元输入分析工具，得到其注释-代码一致性 |
| response               | 工具的输出结果与理想指标的逼近程度            |



# 5. Date collection

我们要研究copy-refactor带来的注释-代码不一致性， 考虑到copy-refactor行为频繁发生在开源项目，我们就选择了四个大型开源项目作为**程序单元的样本**：JDK 8, Guava, Apache Commons Collections , Joda, 这些项目的规模从450到2500个类不等，从43到310个KLOC不等，30%的代码行是注释，这意味着注释在大型项目中占据相当重要的地位，也体现了对注释-代码不一致性研究的重要性。

考虑到程序单元可以对应不同的代码实体: 类、方法和语句块， 而注释编写没有统一的标准， 因此开发者在注释时具有很强的任意性，可以对不同代码实体进行注释。 为了确保研究的泛化性，我们进行了**分层随机抽样**以收集不同代码实体（这对应了不同的程序单元）的注释：类、方法、语句块。对于每个源文件，我们按照文件中每一种代码实体的数量比例随机抽样收集程序单元。这确保了不同种类的程序单元的注释都被覆盖。



# 6. Validity

## 6.1 Construct Validity

对Construct Validity的主要威胁在于：

**(1)** 我们的方法只考虑了注释-代码一致性的最简情况，即代码文本上相等且注释在分类后语义上相等。 事实上代码相等也应该从语义而非文本层面进行判断。

**(2)** 我们采用的注释分类方法( CPC )的实现是有Construct Validity威胁的。 CPC的注释分类以人工分类的数据集作为测试集，对样本进行有监督训练，人工分类过程不可避免地会造成偏见。 

对于(1), 代码的语义等价性判断是十分困难的，该领域现有的工作都没有很好地解决该问题，因此我们只能退而求其次，只计算文本等价性。

对于(2), 为了消除人工分类的偏见，CPC的每条注释都由两名开发人员独立进行分类，当两名开发人员意见不一致时，由第三名开发人员手动解决所有情况。CPC通过测量编码者之间的一致性来评估标签的可靠性.

## 6.2 Internal validity

对于External Validity的主要威胁在于：CPC方法的实现中使用了决策树和卷积网络等易过拟合的机器学习算法。为了降低这个威胁，CPC随机选择了80%的数据集作为训练数据，并采用了ive-fold交叉验证（参见CPC论文）

## 6.3 External validity

对于External Validity的主要威胁在于：

(1) 本研究所选取的样本数量不足，数据量不够大，样本或许也不具有足够的代表性。

(2)  我们对所处理的注释做了简化：我们只研究了Java Doc注释，而非一般的行级和块级注释，这使得我们的研究不够泛化。

对于(1)，未来我们可以选取更多更广泛的样本，来降低外部有效性威胁，提升本方法的泛化能力。

对于(2)，如前所述，完全无结构的注释处理起来十分困难，而只要注释具有很少的结构，比如Java Doc形式，这个问题就可以大大简化， 因此我们认为这种简化是有益的。 