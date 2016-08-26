>https://docs.google.com/document/u/1/d/1RIezQqE4aEhBRmArIAS1mRIZtWFf6JxN_7B4meyWK0Y/pub

接下来的表单改变提议
=================
作者：Kara Erickson, 2016.6.3

这个文件描述“breaking changes only”的计划。
将新增很多表单的改进，特性的请求，bug修改，它们并包含在这个文件中（比如radio buttons,重置表单等等）。这些改变将会在另外一个文件中描述。

已存在的表单API
--------------
### 指令(Directives)：

* ngFormControl
* ngModel
* ngFormModel
* ngControlGroup

### 注射器(Injectables):
* Control()
* ControlGroup()
* ControlArray()
* FormBuilder
* Validators

问题空间（problem space？）
------------------------
###模板驱动的表单：

####示例1：ngControl

	<form #f="ngForm">
  		<div ngControlGroup="name" #name="ngForm">
    		<input ngControl="first" #first="ngForm">
    		<input ngControl="last">
  		</div>
	</form>
	{{ f.value | json }}  // {name: {first: null, last: null}}
	{{ first.value }}    // null

这创建了一个没有初始值的基本表单，你可以输出每个control和controlGroup，去检查它的值(value)和验证状态(vadility state).

问题：
>所有的输出都叫__ngForm__。表面上看来，name,first,f 好像应该指向同一个东西-根表单。实际上，每个__ngForm__是一个基于它的父元素的不同的值。它应该更直观的为每个输出匹配这个指令的名字。

问题：
>ngControl和ngFormControl的区别并不明显。它们都是表单controls(Form controls)，所以为什么用ngControl替代ngFormControl？事实上，ngControl创建了一个新的表单control以及期望一个字符串的名字作为一个输入，而ngFormControl期望传入一个已存在的control实例。另外，只有ngFormControl能被用在Form标签外面。这并不能很明显的从名字中看出。

问题：
>怎么给表单填入初始值不是很明确。你可以作为一个子视图对表单进行查询和手动设置control的值。然而，这不是立即生效的，而且熟悉angular1的开发们更可能试图将它转换成ngModel（示例2）.

目的：
> 提供开发们能轻易猜出的直观的输出名字。
> 
> 区分容易混淆的指令名：ngControl & ngFormControl

####示例2：ngModel
从angular1中过来的开发者期望以下代码能够生效合情合理。在angular1中，添加__ngModel__和一个name属性是向根表单注册表单control全部所需要做的。

	<form #f="ngForm">
   		<div>
     		<input name="first" [(ngModel)]="person.name.first"  
      #first="ngForm">
     		<input name="last" [(ngModel)]="person.name.last">
   		</div>
	</form>
	{{ f.value | json }}  // {}
	{{ first.value }}     // 'Nancy'
	
	class MyComp {
   		person = {
      		name: {
         		first: 'Nancy',
         		last: 'Drew'
     		}     
   		}
	}

问题：
>然而，这个设定只能实现部分效果。angular2不同于angular1，angular2的__ngModel__创建了一个独立的并不知道父组和所属表单的control。但是你可以输出这个独立于表单的control值，也可以设置它的初始值，这个总表单的值是空的。所以检查这个父组或者表单的合法状态也是不可能。你必须通过ngControl得到这个结果。

目的：
>使得迁移到angular2对于angular1的开发者们更直观。

####示例3：ngControl+ngModel
对于开发者而言下一步理所当然就是重复添加__ngControl__和__ngControlGroup__.
   
	<form #f="ngForm">
   		<div ngControlGroup="name">
     		<input ngControl="first" [(ngModel)]="person.name.first"
     #first="ngForm">
     		<input ngControl="last" [(ngModel)]="person.name.last">
   		</div>
	</form>
	{{ f.value | json }}  // {name: {first: 'Nancy', last: 'Drew'}}
	{{ first.value }}     // 'Nancy'
	class MyComp {
   		person = {
      		name: {
         		first: 'Nancy',
         		last: 'Drew'
     		}     
   		}
	}
同时使用这两个，这个表单都能如期工作，填充初始值、设置值、在父表单上正确校验。

问题：
>开发者们第一次看到这段代码，对ngControl在哪里结束和ngModel在哪里开始不是很明确。什么时候你需要用到ngControl，什么时候你需要用到ngModel。这个非常让人困惑，因为ngModel在angular1中有最多的但不是所有的功能，而且这两个指令都为每个独立的controls提供校验状态和校验的class。
>
>这是由于在angular2中我们有两个很相似很容易混淆的的指令，而过去一个指令就足够，并且它们之间的分割线不是很明确。

问题：
>当ngControl和ngModel都在同一个元素上的，哪个作为ngForm的输出不是很明确。事实上，当它们两个同时使用时只有ngControl指令是激活的，而ngModel信息只被作为输入。这个只能通过看源码得知。

目的：
>简化模板驱动的表单设置，避免两个对抗的指令。

"Model-driven forms"

####示例4：ngFormModel

	<form [ngFormModel]="form">
   		<div ngControlGroup="name">
     		<input ngControl="first" #first="ngForm">
     		<input ngControl="last">
   		</div>
	</form>
	{{ form.value | json }}    // {name: {first: 'Nancy', last: 'Drew'}}
	{{ first.value }}         // 'Nancy'
	class MyComp {
   		form = new ControlGroup({
      		name: new ControlGroup({
        		first: new Control('Nancy'),
        		last: new Control('Drew')
     		})
   		});
	}

问题：
>如果你之前都是模板驱动表单，这个看起来就像一份样板文件。你会注意到这个模板很像示例1中的模板。在示例1中，模板已经有__ngControl__和__ngControlGroup__指令，但是它们都创建controls。在这种情况下认为同样的指令创建control合情合理。一个常见的问题是，如果你已经在class中创建了controls，为什么还有在模板中创建？
>
>实际上，你只是简单的把已存在的controls同步到正确的dom元素上。如果你这样认为，那你把每个你在class中创建的control关联到dom元素上确实很有必要。但是指令在模板驱动方式和model驱动方式上有同样的名字，*在这个场景下只是*这个指令，简单用来查找已存在的controls，还是创建它们，不是很明确。

>本质上，我们强迫着两个不同的行为使用同一套指令。如果我们有两套指令在每一种场景下行为都可以预测，就不会那么困惑了。

问题：
>如果这是你看到的第一个表单的示例，看起来创建这个简单表单花费很多工作。如果这是一个高级的用例，那么对于简单的表单，你只需要移除ngFormModel和class的代码。

问题：
>ngFormModel从名字上不好解释清楚。它看起来像ngModel的扩展，可以为整个表单传入一个范畴对象(domain object)。它实际上期望一个ControlGroup，但是并不明显。

目的：
>认清model驱动表单绑定已存在的controls。
>
>由于ngControl/ngControlGroup在模板驱动和model驱动的场景下表现不同，所以在概念上分离开来。
>
>model驱动的表单本来是作为高级用例的，简单的表单可以通过其他方式。

>区分相似容易混淆的指令：ngModel/ngFormModel


####示例5：Validators
	
	class MyComp {
   		form = new ControlGroup({
        	name: new ControlGroup({
        	    first: new Control('Nancy', Validators.required),
           		last: new Control('Drew', Validators.compose(
             	Validators.required, Validators.minLength(3)))
       		})
  		});
	}

这个表单为每个对应的名字添加了校验，last名字有至少三个特性。

问题：
>Controls只能接受一个validator方法，所以第一眼看起来你只能为每个control运行一种校验。不确定你可以将不同的校验（validator）传入compose()，创建一个聚合的校验（validator）。如果我们允许一个数组的校验（validators）在control的结尾组合，这不是一个很好的开发经验。

目的：
>促进将多个validators传入一个control中。


####示例6：Control数组
	<form [ngFormModel]="form">
 		<div ngControlGroup="cities">
    		<div *ngFor="let city of cityArray.controls; let i=index" >
       			<input ngControl="{{i}}">
    		</div>
 		</div>
	</form>
	<button (click)="push()">push</button>
	class myComp {
   		cityArray = new ControlArray([
      		new Control('SF'), new Control('NY')
   		]);
   		form = new ControlGroup({
     		cities: this.cityArray
   		});
   
   		push() {
     		this.cityArray.push(new Control(''));
   		}
	}

ControlArrays允许用户通过push到ControlArray动态的添加表单controls。

问题：
>你可以使用ngControlGroup指令来将一个ControlArray同步到一个dom元素上。期望ControArray和ngControlArray一样在这个dom上达到同样的效果。

目的：
>为ControlArray创建一个更精简的实践方式。


提议的API改变
-----------
模板驱动/angular1风格

提示-目标：

* 为输出提供一套开发者能轻易猜出的直观的名字
* 使得对angular1开发者而言迁移到angular2表单更直观
* 简化模板驱动表单程序，避免两种对抗的指令

在目前的表单模块中，我们有两个非常相似的指令（ngControl和ngModel），两个提供了75%共同的行为。为了得到100%的行为，目前你不得不同时使用它们两个。同时维护两个几乎拥有完全相同的功能的两个独立的指令不合理，所以这需要简化和合并。

因为大多数angular开发者已经熟悉ngModel，准许ngModel拥有25%的功能，由于重复去除ngControl，比较合理。这使得ngModel在功能与angular1的对等，有助于开发者从angular1中迁移过来。更重要的是，开发者不愿意不得不纠结怎么结合两个相似的指令，他们一直以来都知道使用ngModel。

这也简化了输出的名字的问题，因为就只有一个概念：ngModel。


注意到这个改变不再要求开发者们使用双向绑定很重要。这样就只有一个选择。ngModel也可以被用作单向绑定，而且同ngControl一样值的改变将会通知范例（value change subscription paradigms common with ngControl.）。

为了依照ngModel的规则，确保很容易被记住，ngControlGroup将会被重命名ngModelGoup。从命名中可以明显看出这两个会形成一对，且可以一起使用。如果你想校验一个独立的输入框，使用ngModel，如果你想校验多个输入框，使用ngModelGroup（如下示例）。

### API:
* ngModel		(same)
* ngModelGroup  (deprecated:ngControlGroup)
* --			(deprecated:ngControl)

### 输出（Exports）：
* <form\>		ngForm			(same)
* ngModel		ngModel			(deprecated:ngForm)
* ngModelGroup	ngModelGroup	(deprecated:ngForm)

为了解决猜测获取指令示例的工作，输出的名字总是与指令的名字有对应关系。

####简单的ngModel示例：

	<input [(ngModel)]="person.name.first" #first="ngModel">
	<input [(ngModel)]="person.name.last">
	{{ first.valid }}  // true
	class MyComp {
   		person = {
      		name: {
       		  	first: 'Nancy',
         		last: 'Drew'
     		}     
   		}	
	}

这是一个简单的示例。无论有没有一个包含的form标签，ngModel都可以使用。如果你想孤立表单controls，他们可以独立校验。

####简单的form示例：

	<form #f="ngForm">
  		<input name="first" [(ngModel)]="person.name.first">
  		<input name="last" [(ngModel)]="person.name.last">
	</form>
	{{ f.value }}  // {first: 'Nancy', last: 'Drew'}
	class MyComp {
   		person = {
      		name: {
         	first: 'Nancy',
         	last: 'Drew'
    	 	}     
   		}
	}

和angular1一样，如果你想注册ngModel到父表单，name属性是必须的。这个name在父表单中被用作这个表单control值（value）的关键字（key）。

####没有双向绑定的示例（填充初始值）

有时你想创建一个填充初始值的简单表单，但是不想通过双向绑定。这个例子中，只使用了中括号。

	<form #f="ngForm">
  		<input name="first" [ngModel]="person.name.first">
  		<input name="last" [ngModel]="person.name.last">
	</form>
	{{ f.value }}  // {first: 'Nancy', last: 'Drew'}
	class MyComp {
   		person = {
      		name: {
         		first: 'Nancy',
         		last: 'Drew'
     		}     
   		}
	}

####没有初始值的绑定示例

通常你会使用ngControl实现这个，但是你可以通过name属性和一个未绑定的ngModel指令达到同样的效果。

	<form #f="ngForm">
  		<input name="first" ngModel>
  		<input name="last" ngModel>
	</form>
	{{ f.value }}  // {first: '', last: ''}

从这段设置中可以清晰的看出，ngModel是一个授予校验状态和向表单注册的一个实体。没有它，你只是有个正常的html input。

如果稍后你想为他添加双向绑定，你可以很轻松的将ngModel改成[(ngModel)] (banana box notation)。不需要其他工作，更重要的是，来回切换不需要任何新的概念。

####有子组的校验示例

	<form #f="ngForm">
  		<div ngModelGroup="name">
     		<input name="first" ngModel>
     		<input name="last" ngModel>
  		</div>
	</form>
	{{ f.value }}    // {name: {first: '', last: ''}}

和ngControlGroup一样，ngModelGroup在hood（译者注：？）下创建一个FormGroup实例以及把它连接到元素上。

####有自定义表单control的示例

当一个用户创建一个自定义表单，而且已经在内部为了其他事情使用了这个name属性，我们仍想控制这个案例。

在这个特殊的情况下，你可以通过ngModelOptions传递这个表单control的名字，然后这个名字会被用于向父表单注册这个自定义control（例子中，自定义名字是“first”，所以f.value.first是已定义的）。最后，跟angular1一样，ngModelOptions将包含更多设置，更多功能。

	<form #f="ngForm">
		 <person-input name="Nan" [ngModelOptions]="{name: 'first'}" ngModel>
	</form>
	{{ f.value }}  // {first: ''}

####有radio按钮的示例


	<form #f="ngForm">
 		<input type="radio" name="food" [(ngModel)]="food" value="one">
 		<input type="radio" name="food" [(ngModel)]="food" value="two">
	</form>
	{{ f.value | json }}    // {food: 'one'}

	class MyComp {
  		food = 'one';
	}



模型驱动（Model-driven）/Reactive form
------------------------

提示--目标

* 阐明模型驱动表单绑定已存在的controls
* 概念上区分在模板驱动和模型驱动场景下的ngControl/ngControlGroup，它们表现不同
* 破除ngModel和ngFormModel这两个相似的命令的混淆
* 破除ngControl和ngFormControl这两个相似命令的混淆
* 传达模型驱动表单打算是用来作为高级用例，简单的表单应该使用其他方式


###API：

* formGroup                     (deprecated: ngFormModel)
* formControl                   (deprecated: ngFormControl)
* formControlName      			(deprecated: ngControl)
* formGroupName              	(deprecated: ngControlGroup)
* formArrayName                 (deprecated: ngControlGroup)
* FormControl()              	(deprecated: Control)
* FormGroup()                  	(deprecated: ControlGroup)
* FormArray()               	(deprecated: ControlArray)
* FormBuilder              		(same)
* Validators                    (same)


####“form”前缀

我们将两种表单策略分离成两套指令

* 模板驱动/angular1类型 指令都被作为平台指令
* 这意味着ngModel和ngModelGroup默认可用
* 由于它们都是平台指令，它们可以使用核心ng前缀
* 模型驱动/reactive form 指令作为@angular/common的一部分，必须手动导入
* 作为手动导入指令选择加入，它们抛弃ng前缀。它们使用form前缀来代替，这样直到它们都关联到表单它们才组成了一个整体。

分离有助于阐明哪个指令属于哪个方法，也强调了在开始一个项目的时候reactive指令不是必须的。默认指令组合成一个全功能的块，手动导入的指令组合成一个全功能的块，利用不同的策略。

####FormControl，FormGroup，FormArray

Control，ControlGroup，和ControlArray将被重命名为FormControl，FormGroup和FormArray，由于以下一些原因：

* 这些名字更容易立即识别出，以及更好的描述了他们代表了什么类。Control自己可以描述不同的东西。
* 通过form前缀使得它们与对应的指令更加相似

####formGroup，formControl

ngFormModel改名为formGroup，因为它期望一个FormGroup()实例作为输入，这样更容易产生联系。这个名字也更容易区分于ngModel，防止与ngModel相关产生任何混淆。

我们将bindFormGroup作为一个可选名字，但是对应的kebab-case（译者注：？）“bind-bind-form-froup”将会太难以理解。

formControl是formGroup的搭配指令，用于绑定已存在的单独的control。

####formControlName / formGroupName

旧的API有一个问题，什么时候创建controls，什么时候已存在的controls和dom搭配上不是很明确。我们也想区分这些指令，期望FormControl或者FormGroup的实例和指令们能简单的接收一个control的名字。有了name后缀，formControlName和formGroupName显然期待一个字符串的输入。这些名字也使得与ngModelGroup（它们确实创建了自己的control）区分开来。

####没有输出

由于最好的方法应该是在内部从class这边使用值，我们不想提供输出。在这里允许输出将会混淆我们期望如何reactive表单和已存在的模板/模型分离问题。

####绑定已存在的form示例：

	<form [formGroup]="myForm">
  		<div formGroupName="name">
    		<input formControlName="first">
    		<input formControlName="last">
  		</div>
	</form>
	{{ myForm.value | json }}   // {name: {first: 'Nancy', last: 'Drew'}}
	class MyComp {
   		myForm = new FormGroup({
      		name: new FormGroup({
         	first: new FormControl('Nancy'),
         	last: new FormControl('Drew')
           })
   		});
	}

###异步单向绑定已存在的表单

有事你想通过异步资源获取数据，比如http，所以你不可能数据跟表单一起初始化。在这种情况下，你可以调用__this.form.setValue()__或者__this.form.patchValue()__，而不是使用ngModel。

	<form [formGroup]="myForm">
   		<div formGroupName="name">
     		<input formControlName="first">
     		<input formControlName="last">
   		</div>
	</form>
	{{ myForm.value | json }}   // {name: {first: 'Nancy', last: 'Drew'}}
	class MyComp implements OnInit {
   		person = {
     		name: {first: '', last: ''}
   		};
   		myForm = new FormGroup({
      		name: new FormGroup({
         	first: new FormControl(),
         	last: new FormControl()
      	   })
   		});
  		constructor(private _http: Http)
  		ngOnInit() {
     		this._http.get(someUrl)
            	  .map(this.extractData)
            	  .subscribe(person => this.myForm.patchValue(person));
  		}
	}

####校验示例

	class MyComp {
   		form = new FormGroup({
      		name: new FormGroup({
         		first: new FormControl('Nancy', Validators.required),
         		last: new FormControl('Drew', [
           			Validators.required,
           			Validators.minLength(3)
         		])
     		})
   		});
	}

Non-breaking：FormControl()接受一个合成的validator方法或者一个validators数组，它自己知道如何将validator数组合成。

那样的话，你可以自己用一种特殊的方式合成validators，也可以简单的传入多个validator。

####FormArray示例
	<form [formGroup]="form">
 		<div formArrayName="cities">
    		<div *ngFor="let city of cityArray.controls; let i=index" >
       			<input [formControlName]="i">
    		</div>
 		</div>
	</form>
	<button (click)="push()">push</button>

	class myComp {
   		cityArray = new FormArray([
     		new FormControl('SF'), new FormControl('NY')
  		]);
   		form = new FormGroup({
     		cities: this.cityArray
   		});
   
  		 push() {
  			   this.cityArray.push(new FormControl(''));
  		 }
	}

两件事情：

* 使用FormArray的时候，使用formArrayName比使用ngControlGroup要更好。
* Non-breaking:这是一个除了保存原始control数组之外简单遍历controls的方式。

####导入示例

	import {REACTIVE_FORM_DIRECTIVES, FormControl, FormGroup} from '@angular/forms';
	…
	directives: [REACTIVE_FORM_DIRECTIVES]

###TLDR：问题

* 所有的表单指令导出为ngForm很让人困惑
* 从__ngControl/ngFormControl__和__ngModel/ngFormModel__的名字看它们区别不是很清楚。
* angular1开发们会惊讶于ngModel只有它原有功能的75%
* 在模板驱动方式中如何给表单填充初始值不是很明确
* 什么时候使用ngControl什么时候使用ngModel和它们分别做了什么，不是很明确
* __ngControl、ngControlGroup__在模板驱动和模型驱动场景下表现不同，让人很困惑。
* 模型驱动指令的命名不能传达已存在的controls会被同步，而不是被创建两次。
* validator的名字都不直观
* 如何将多个validator传入一个control中
* ControlArray指令的名字

###TLDR：目的

* 提供一套直观的开发者们能轻易猜出的输出名字
* 区分 像 ngControl/ngFormControl和ngForm/ngFormModel 这种很相似又容易混淆的指令。
* 使angular1开发者们迁移到angular2表单更直观
* 简化模板驱动表单，拒绝两种对抗的指令
* 阐明模型驱动表单绑定已存在的controls
* 由于在模板驱动和模型驱动两种场景下ngControl/ngControlGroup行为不同，从概念上分离它们
* 传达 模型驱动表单计划作为高级用例使用，而简单点的表单可以使用其他方式创建
* 保证validator的名字都很直观，基于对应的dom。
* 促使将多个validators传入一个control
* 创建一个简化的ControlArrays
* 定义一个可以有助于阻止故障和阻止开发者沮丧的弃用策略（译者注：已崩溃。。）

###TLDR:API名

####平台指令

* ngModel                 		(same)  
* ngModelGroup                  (deprecated: ngControlGroup)

####从@angular/form导入
REACTIVE_FORM_DIRECTIVES:

* formGroup                      (deprecated: ngFormModel)
* formControl                	 (deprecated: ngFormControl)
* formControlName                (deprecated: ngControl)
* formGroupName                	 (deprecated: ngControlGroup)
* formArrayName                  (deprecated: ngControlGroup)
* FormControl()                  (deprecated: Control)
* FormGroup()                	 (deprecated: ControlGroup)
* FormArray()                	 (deprecated: ControlArray)
* FormBuilder               	 (same)
* Validators                     (same)

###弃用策略

新的表单代码在它自己的模块@angular/forms中

RC-2：

使用弃用表单：无变化

切换到新表单：

* 使用__disableDeprecatedForms()__去禁用旧表单平台指令
* 使用__provideForms()__方法提供新的表单平台指令
* 从__@angular/forms__而不是__@angular/common__中导入任何需要的符号

bootstrap文件示例：

	import {disableDeprecatedForms, provideForms} from '@angular/forms';
	bootstrap(AppComponent, [
   		disableDeprecatedForms()
   		provideForms()
	])

RC-5：

默认状态：没有表单

使用新表单：

* 提供FormsModule（或者调用provideForms()-该方法在下个RC消失）
* 从@angular/forms导入任何需要的符号

使用废弃表单：
* 提供DeprecatedFormModule
* 从@angular/common导入任何需要的符号

bootstrap文件示例（使用新表单）：


	import {FormsModule} from '@angular/forms';
	bootstrap(AppComponent, {modules: [FormsModule] })
 
bootstrap文件示例（使用废弃表单）：


	import {DeprecatedFormsModule} from '@angular/common';
	bootstrap(AppComponent, {modules: [DeprecatedFormsModule] })

或者你可以选择不在bootstrap时而是在需要时添加FORM_DIRECTIVE和FORM_PROVIDERS：


	import {FORM_DIRECTIVES, FORM_PROVIDERS} from '@angular/forms';
	@Component({
  		selector: 'some-comp',
  		template: '',
  		directives: [FORM_DIRECTIVES],
  		providers: [FORM_PROVIDERS]
	})
	class SomeComp {}