# Լամբդա լեզվի իրականացում

_Փորձեր JavaScript-ի և Node.js-ի հետ_

JavaScript-ը հերթական ծրագրավորման լեզուն է, որի ուսումնասիրությամբ որոշեցի
զբաղվել վերջին մի քանի շաբաթների հանգստյան օրերին։ Եվ ինչպես միշտ նոր լեզվի
ուսումնասիրությունը սկսում եմ մի որևէ այլ լեզու _իրականացնելու_ համար։ Այս
անգամ որպես իրականացվող լեզու ընտրել եմ պարզագույն _Լամբդա_ լեզուն։ Ահա դրա
քերականությունը.

```
expression
    = REAL
    | IDENT
    | '(' expression ')'
    | BUILTIN expression+
    | 'if' expression 'then' expression 'else' expression
    | 'lambda' IDENT+ ':' expression
    | 'apply' expression 'to' expression
    .
```

Այստեղ _իրական_ թվերն են, _փոփոխականները_, _խմբավորման_ փակագծերը, լեզվի
_ներդրված_ գործողությունները, _պայմանական_ արտահայտությունը, ինչպես նաև
_աբստրակցիայի_ ու _ապլիկացիայի_ գործողությունները։ Ֆունկցիոնալ ծրագրավորման
տեսությունից հայտնի է, որ այսքանը բավական է Լամբդա լեզուն ոչ միայն որպես
ընդլայնված հաշվարկիչ օգտագործելու, այլ նաև լիարժեք (թվային) ալգորիթմներ
կազմելու համար։


## Վերլուծություն

Լամբդա լեզվով գրված տեքստի վերլուծության `parser.js` մոդուլը արտաքին
աշխարհին տրամադրում է (exports) միակ `parse` ֆունկցիան։ Վերջինս արգումենտում
ստանում է վերլուծվող տեքստը և վերադարձնում է _աբստրակտ քերականական ծառը_։

Նախ՝ տեքստը տրոհվում է _լեքսեմների_ (lexeme) ցուցակի՝ միաժամանակ ամեն մի
լեքսեմին կապելով համապատասխան _պիտակը_ (token)։ Այնուհետև շարահյուսական
վերլուծիչը, օգտագործելով լեքսեմների ցուցակը, կառուցում է աբստրակտ
քերականական ծառը։

Տեքստը լեքսեմների ցուցակի տրոհող `scanOne` և `scanAll` ֆունկցիաները գրել եմ
ֆունկցիոնալ մոտեցմամբ։ `scanOne` ֆունկցիան արգումենտում ստանում է տեքստ, և
վերադարձնում է եռյակ՝ տեքստի սկզբից «պոկված» լեքսեմը, դրա պիտակը և տեքստի
չտրոհված մասը։ Օրինակ, `scanOne('if + a b then a else b)` կանչի արժեքն է
`{ token: 'IF', value: 'if', rest: ' + a b then a else b'}` օբյեկտը։ Տեքստից
ինձ հետաքրքրող մասը պոկում եմ կանոնավոր արտահայտություների օգնությամբ։
Ահա `scanOne` ֆունկցիան՝ համապատասխան մեկնաբանություններով.

```JavaScript
// Լեզվի ծառայողական բառերի ցուցակը
const keywords = ['if', 'then', 'else', 'lambda', 'apply', 'to']

// Տեքստից կարդալ մեկ (թոքեն, լեքսեմ) զույգ
var scanOne = function(text) {
    // տեքստի վերջը
    if( text == '' ) {
        return { token: 'EOS', value:'EOS', rest: '' }
    }

    // անտեսել բացատանիշերը
    let mc = /^[ \n\t\r]+/.exec(text)
    if( mc != null ) {
        return scanOne(text.substring(mc[0].length))
    }

    // ծառայողական բառեր և իդենտիֆիկատորներ
    mc = /^[a-zA-z][0-9a-zA-z]*/.exec(text)
    if( mc != null ) {
        return {
            token: keywords.includes(mc[0]) ? mc[0].toUpperCase() : 'IDENT',
            value: mc[0],
            rest: text.substring(mc[0].length)
        }
    }

    // իրական թվեր
    mc = /^[0-9]+(\.[0-9]+)?/.exec(text)
    if( mc != null ) {
        return {
            token: 'REAL',
            value: mc[0],
            rest: text.substring(mc[0].length)
        }
    }

    // ծառայողական սիմվոլներ (մետասիմվոլներ)
    mc = /^(\(|\)|:)/.exec(text)
    if( mc != null ) {
        return {
            token: mc[0],
            value: mc[0],
            rest: text.substring(mc[0].length)
        }
    }

    // գործողությունների նշաններ
    mc = /^(\+|\-|\*|\/|=|<>|>|>=|<|<=)/.exec(text)
    if( mc != null ) {
        return {
            token: 'OPER',
            value: mc[0],
            rest: text.substring(mc[0].length)
        }
    }

    // չնախատեսված, չսպասված նիշ
    return { token: 'UNKNOWN', value: text[0], rest: text }
}
```

Իսկ `scanAll` ֆունկցիան կանչում է `scanOne` ֆունկցիան այնքան ժամանակ, քանի
դեռ հերթական կանչի արդյունքում չի ստացվել `token == 'EOS'` օբյեկտ։

```JavaScript
// Կարդալ բոլոր (թոքեն, լեքսեմ) զույգերն ու վերադարձնել ցուցակ
var scanAll = function(text) {
    let res = []
    let ec = scanOne(text)
    while( ec.token != 'EOS' ) {
        res.push({token: ec.token, value: ec.value})
        ec = scanOne(ec.rest)
    }
    res.push({token: 'EOS', value: 'EOS'})
    return res
}
```

Այս երկու ֆունկցիաները կազմում են Լամբդա լեզվի բառային վերլուծիչը։ Հիմա՝
շարահյուսական վերլուծության մասին։

`parse` ֆունկցիան `scanAll` ֆունկցիայով տրոհում է իր արգումենտում ստացված
ծրագիրը և լեքսեմների ցուցակը վերագրում է `lexemes` գլոբալ սիմվոլին։ `index`
գլոբալ հաշվիչին վերագրելով `0` արժեքը՝ նշվում է, որ վերլուծությունը
սկսվելու է ցուցակի առաջին լեքսեմից։ Եվ վերադարձնում է `expression`
ֆունկցիայի կանչի արժեքը, որն ինքը հենց Լամբդա լեզվի վերլուծիչն է։

```JavaScript
// (թոքեն, լեքսեմ) զույգերի ցուցակ
var lexemes = []
// ընթացիկ օգտագործվող տարր ինդեքսը
var index = 0;

// ծրագրի տեքստի վերլուծություն
var parse = function(text) {
    lexemes = scanAll(text)
    index = 0
    return expression()
}
```

Բայց, մինչև `expression`-ին անցնելը, մի քանի օգնական ֆունկցիաների մասին։
`have` ֆունկցիան վերադարձնում է `true`, եթե լեքսեմների ցուցակի `index`-րդ
տարրի պիտակը հավասար է արգումենտում տրված պիտակին կամ պիտակներից որևէ
մեկին։ Այս ֆունկցիայի արգուենտը կարող է լինել ինչպես առանձին պիտակ,
այնպես է պիտակների վեկտոր։

```JavaScript
// ստուգել ցուցակի ընթացիկ տարրը
var have = function(exp) {
    let head = lexemes[index].token

    if( exp instanceof Array )
        return exp.includes(head)

    return head == exp
}
```

Հաջորդ, `next` ֆունկցիան մեկով ավելացնում է լեքսեմների ինդեքոը՝ դիտարկվելի
դարձնելով լեքսեմների ցուցակի հաջորդ տարրը։ Բայց վերադարձնում է ցուցակի
նախորդ տարրի արժեքը՝ `value` սլոթի արժեքը։

```JavaScript
// անցնել հաջորդին, և վերադարձնել նախորդի արժեքը
var next = function() {
    return lexemes[index++].value
}
```

`match` ֆունկցիան համադրում է `have` և `next` ֆունկցիաները. եթե լեքսեմների
ցուցակի հերթական դիտարկվող տարրի պիտակը հավասաար է `match`-ի արգումենտին,
ապա դիտարկելի դարձնել հաջորդ տարրը։ Եթե հավասար չէ, ապա ազդարարվում է
շարահյուսական սխալի մասին։

```JavaScript
// ստուգել և անցնել հաջորդին
var match = function(exp) {
    if( have(exp) )
        return next()
    throw 'Syntax error.'
}
```

`expression` ֆունկցիայի կառուցվածքը ուղղակիորեն արտացոլում է այս գրառման
սկզբում բերված քերականությանը։ Ինչպես քերականությունն աջ մասն է բաղկացած
յոթ այլընտրանքներից (տարբերակներից), այնպես էլ `expression`  ֆունկցիան է
կազմված յոթ տրամաբանական հատվածներից։ Ամեն մի հատվածը ձևավորում ու
վերադարձնում է աբստրակտ քերականական ծառի մի որևէ հանգույց։ Այդ
հանգույցներն ունեն `kind` սլոթը, որով որոշվում է հանգույցի տեսակը։
Ստորև բերված է `expression` ֆունկցիան՝ մանրամասն մեկնաբանություններով.

```JavaScript
// Լամբդա լեզվի արտահայտությունները կարող են սկսվել միայն հետևյալ
// պիտակներով։ Գրականության մեջ այս բազմությունը կոչվում է FIRST.
// FIRST(expression)
const exprFirst = ['REAL', 'IDENT', '(', 'OPER', 'IF', 'LAMBDA', 'APPLY']

// Արտահայտությունների վերլուծությունը
var expression = function() {
    // եթե դիտարկվող լեքսեմը իրական թիվ է,
    // ապա վերադարձնել AST-ի հանգույց, որի
    // տիպը REAL է
    if( have('REAL') ) {
        let vl = next()
        return { kind: 'REAL', value: parseFloat(vl) }
    }

    // եթե լեքսեմը իդենտիֆիկատոր է, ապա կառուցել
    // փոփոխականի (անուն) հղում ներկայացնող հանգույց
    if( have('IDENT') ) {
        let nm = next()
        return { kind: 'VAR', name: nm }
    }

    // եթե լեքսեմը բացվող փակագիծ է, ապա վերադարձնել
    // փակագծերի ներսում գրված արտահայտության ծառը
    if( have('(') ) {
        next()
        let ex = expression()
        match(')')
        return ex
    }

    // Լամբդա լեզվի օգտագործումը մի քիչ ավելի հեշտացնելու
    // համար ես դրանում ավելացրել եմ ներդրված գործողություններ։
    // դրանք պրեֆիքսային են, ինչպես Լիսպում՝ ցուցակի առաջին
    // տարրը գործողության նիշն է, որը կարող է լինել թվաբանական,
    // համեմատման կամ տրամաբանական գործողություն
    if( have('OPER') ) {
        // վերցնել գործողության նիշը
        let op = next()
        // վերլուծել առաջին արտահայտությունը
        let args = [ expression() ]
        // քանի դեռ հերթական լեքսեմը պատկանում է FIRST(expression)
        // բազմությանը, վերլուծել հաջորդ արտահայտությունը
        while( have(exprFirst) )
            args.push(expression())
        // կառուցել լեզվի ներդրված գործողության հանգույցը
        return { kind: 'BUILTIN', operation: op, arguments: args }
    }

    // պայմանական արտահայտությունը բաղկացած է if, then, else
    // ծառայողական բառերով բաժանված երեք արտահայտություններից
    if( have('IF') ) {
        next()
        // վերլուծել պայմանի արտահայտությունը
        let co = expression()
        match('THEN')
        // վերլուծել պայմանի ճիշտ լինելու դեպքում
        // հաշվարկվող արտահայտությունը
        let de = expression()
        match('ELSE')
        // պայմանի կեղծ լինելու դեպքում հաշվարկվող
        // արտահայտությունը
        let al = expression()
        // պայմանակա արտահայտության հանգույցը
        return { kind: 'IF', condition: co, decision: de, alternative: al }
    }

    // անանուն ֆունկցիայի սահմանումը սկսվում է lambda
    // բառով, որին հաջորդում են ֆունկցիայի պարամետրերը,
    // (ֆունկցիան պիտի ունենա գոնե մեկ պարամետր), հետո,
    // «:» նիշից հետո ֆուկցիայի մարմինն է
    if( have('LAMBDA') ) {
        next()
        // պարամետրերը
        let ps = [ match('IDENT') ]
        while( have('IDENT') )
            ps.push(next())
        match(':')
        // մարմինը
        let by = expression()
        // անանուն ֆունկցիայի հանգույցը
        return { kind: 'LAMBDA', parameters: ps, body: by, captures: [] }
    }

    // apply գործողությունը իրեն հաջորդող արտահայտությունը
    // կիրառում է to բառից հետո գրված արտահայտություններին
    if( have('APPLY') ) {
        next()
        // վերլուծել կիրառելի աարտահայտությունը
        let fn = expression()
        match('TO')
        // վերլուծել արգումենտները
        let args = [ expression() ]
        while( have(exprFirst) )
            args.push(expression())
        // ֆունկցիայի կիրառման հանգույցը
        return { kind: 'APPLY', callee: fn, arguments: args }
    }

    // բոլոր այլ դեպքերում ազդարարել շարահյուսական սխալի մասին
    throw 'Syntax error.'
}
```

Վերջում նշեմ, որ Լամբդա լեզվի վերլուծիչն իրականացրել եմ _ռեկուրսիվ վայրէջքի_
եղանակով։ Այդ մասին կարելի է կարդալ ծրագրավորման լեզուների իրականացմանը
նվիրված ցանկացած գրքում։


## Աբստրակտ քերականական ծառը

Լամբդա լեզվով գրված ծրագրի վերլուծության արդյունքում կառուցվում է աբստրակտ
քերականական ծառ, որի հանգույցների տեսակը որոշվում է `kind` սլոթով։ Օրինակ,
`parse('3.14')` կիրառման արդյունքում կառուցվում է `{ kind: 'REAL', value: 3.14 }`
օբյեկտը, որի `kind` սլոթի `REAL` արժեքը ցույց է տալիս, որ սա իրական թիվ
ներկայացնող հանգույց է, իսկ `value` սլոթի արժեքն էլ թվի մեծությունն է։

Մեկ այլ օրինակ, `parse('+ 3.14 x')` ծրագրի վերլության արդյունքում կառուցվում
է հետևյալ օբյեկտը.

```JavaScript
{ kind: 'BUILTIN',
  operation: '+',
  arguments: [ { kind: 'REAL', value: 3.14 }, { kind: 'VAR', name: 'x' } ] }
```

Այստեղ հանգույցի տեսակը `BUILTIN` է (լեզվի ներդրված գործողություն),
գործողության տեսակը՝ `operation`, գումարումն է, արգումենտների վեկտորն էլ
պարունակում է երկու օբյեկտ՝ առաջինը իրկան թիվ ներկայացնող հանգույց է,
իսկ երկրորդը փոփոխականի հղում ներկայացնող հանգույց։

`lambda x : * x x` լամբդա արտահայտության վերլուծության արդյունքում
կառուցվում է մի օբյեկտ, որում `kind == 'LAMBDA'`,  պարամետրերի ցուցակը
պարունակում է միայն `x` փոփոխականի անունը, իսկ մարմինը բազմապատկման
ներդրված գործողությունը ներկայացնող հանգույց է (`captures` սլոթի մասին
կխոսեմ լամբդա արտահայտությունների ինտերպրետացիայի բաժնում)։

```JavaScript
{ kind: 'LAMBDA',
  parameters: [ 'x' ],
  body:
   { kind: 'BUILTIN',
     operation: '*',
     arguments: [ [Object], [Object] ] },
  captures: {} }
```


## Ինտերպրետացիա

Լամբդա ծրագրի վերլուծության արդյունքում կառուցված ծառի _ինտերպրետացիայի_
`evaluate` ֆունկցիան նույնպես կառուցված է ռեկուրսիվ սխեմայով։ Դր առաջին
արգումենտը ծրագրի աբստրակտ քերականական ծառն է, իսկ երկրորդը՝ հաշվարկման
միջավայրը։ Վերջինս մի արտապատկերում է (`map`), որում փոփոխականներին
համապատասխանեցված են ընթացիկ արժեքները։ Քանի որ Լամբդա լեզվում վերագրման
գործողություն չկա, փոփոխականներին արժեքներ կարող են կապվել ֆունկցիայի
պարամետրերի օգնությամբ։

```JavaScript
var evaluate = function(expr, env) { /* ... */ }
```

Ինչպես երևում է `expression` ֆունկցիայից, վերլուծության արդյուքնում
կառուցվում են վեց տեսակի հանգույցներ. `REAL`, `VAR`, `BUILTIN`, `IF`,
`LAMBDA` և `APPLY`։ `evaluate` ֆունկցիայում դիտարկվում են այս վեց դեպքերը։
Հիմա ես հերթով ու հնարավորինս մանրամասն կներկայացնեմ նշված վեց հանգույցների
հաշվարկման եղանակները։

`REAL` տիպի հանգույցի հաշվարկման արդյունքը դրա `value` սլոթի արժեքն է։

```JavaScript
if( expr.kind == 'REAL' ) {
    return expr.value
}
```

`VAR` տիպի հանգույցի հաշվարկման արժեքը ստանալու համար միջավայրից
վերադարձնում եմ `name` սլոթին կապված արժեքը։

```JavaScript
if( expr.kind == 'VAR' ) {
    return env[expr.name]
}
```

`BUILTIN` տիպի հանգույցի արժեքը ստանալու համար պետք է նախ հաշվարկել
`arguments` ցուցակի արտահայտությունների արժեքները, ապա գրանց նկատմամբ
կիրառել `operation` սլոթում գրանցված գործողությունը։

```JavaScript
if( expr.kind == 'BUILTIN' ) {
    let evags = expr.arguments.map(e => evaluate(e, env))
    return evags.reduce(builtins[expr.operation])
}
```

`IF` տիպի հանգույցը, որ պայմանական արտահայտության մոդելն է, հաշվարկելու
համար նախ հաշվարկվում է `condition` սլոթի արժեքը՝ պայմանը։ Եթե այն տարբեր
է `0.0` թվային արժեքից՝ _ճշմարիտ_ է, ապա հաշվարկվում և վերադարձվում է
`decision` սլոթի արժեքը։ Եթե `condition`-ի արժեքը զրո է, ապա հաշվարկվում ու
վերադարձվում է `alternative` սլոթին կապված արտահայտության արժեքը։

```JavaScript
if( expr.kind == 'IF' ) {
    let co = evaluate(expr.condition, env)
    if( co !== 0.0 )
        return evaluate(expr.decision, env)
    return evaluate(expr.alternative, env)
}
```

`LAMBDA` տիպի հանգույցի հաշվարկման արդյունքում պիտի կառուցվի մի օբյեկտ,
որը կոչվում է _closure_ (չգիտեմ, թե հայերեն սրան ինչ են ասում)։ Իմաստն այն
է, որ `LAMBDA` օբյեկտի `captures` սլոթում գրանցվում են `body` սլոթին կապված
արտահայտության _ազատ փոփոխականների_ արժեքները՝ հաշվարկված ընթացիկ
միջավայրում։ Այս կերպ լրացված `LAMBDA` օբյեկտն արդեն հնարավոր կլինի `apply`
գործողության կիրառել արգումենտների նկատմամբ։ (Արտահայտության մեջ մտնող
ազատ փոփոխականների բազմությունը հաշվարկող `freeVariables` ֆունկցիայի մասին
քիչ ավելի ուշ)։

```JavaScript
if( expr.kind == 'LAMBDA' ) {
    let clos = Object.assign({}, expr)
    let fvs = freeVariables(clos)
    for( let v of fvs )
        clos.captures[v] = env[v]
    return clos
}
```

Մի օրինակ. թող որ տրված է `lambda y : + x y` արտահայտությունը և `{ 'x': 7 }`
հաշվարկման միջավայրը։ Ինչպես արդեն նշեցի վերլուծության մասին պատմելիս,
այս տրված ծրագրի վերլուծությունը կառուցելու է այսպիսի մի օբյեկտ.

```JavaScript
{ kind: 'LAMBDA',
  parameters: [ 'y' ],
  body:
   { kind: 'BUILTIN',
     operation: '+',
     arguments: [ [Object], [Object] ] },
  captures: {} }
```

Երբ այս օբյեկտը հաշվարկում եմ `{ 'x': 7 }` միջավայրում, ստանում եմ նույն
օբյեկտը, բայց արդեն լրացված `captures` սլոթով։

```JavaScript
{ kind: 'LAMBDA',
  parameters: [ 'y' ],
  body:
   { kind: 'BUILTIN',
     operation: '+',
     arguments: [ [Object], [Object] ] },
  captures: { x: 7 } }
```

`APPLY` հանգույցի հաշվարկումն ամենահետաքրքիրն ու ամենակարևորն է։ Նախ պետք է 
հաշվարկել կիրառվող ֆունկցիան՝ `apply` բառից հետո գրած արտահայտությունը, ու 
համոզվել, որ ստացվել է `LAMBDA` տիպի օբյեկտ։ Ավելի ճիշտ, պետք է սատացվի closure,
որի `captures`-ը պարունակում է լամբդայի մարմնի ազատ փոփոխականների արժեքները
(bindings)։ 
Հետո պետք է հաշվարկել `APPLY` օբյեկտի `arguments` սլոթին կապված ցուցակի 
արտահայտություններն, ու, դրանք համադրելով closure-ի պարամետրերին, ստանալ 
պարամետր—արժեք արտապատկերում։

```JavaScript
if( expr.kind == 'APPLY' ) {
    let clos = evaluate(expr.callee, env)
    if( clos.kind != 'LAMBDA' )
        throw 'Evaluation error.'
    let nenv = Object.assign({}, clos.captures)
    let evags = expr.arguments.map(e => evaluate(e, env))
    let count = Math.min(clos.parameters.length, evags.length)
    for( let k = 0; k < count; ++k )
        nenv[clos.parameters[k]] = evags[k]
    return evaluate(clos.body, nenv)
}
```



## Read-Eval-Print Loop


## Աղբյուրներ

Ֆունկցիոնալ լեզվի իրականացման հարցերը քննարկվում են շատ գրքերում ու
հոդվածներում։ Ես անհրաժեշտ եմ համարում դրանցից մի քանիսի թվարկումը.

1. Christian Queinnec, _Lisp in Small Pieces_, Cambridge University Press, 2003.
2. Peter Norvig, _Paradigms of Artificial Intelligence Programming: Case Studies in Common Lisp_,  Morgan Kaufmann, 1991.
3. Harold Abelson, Jerald Jay Sussman, Julie Sussman, _Structure and Interpretation of Computer Programs_, 2nd Edition, MIT Press, 1996.
4. Peter Norvig, _[(How to Write a (Lisp) Interpreter (in Python))](http://norvig.com/lispy.html)_ և _[(An ((Even Better) Lisp) Interpreter (in Python))](http://norvig.com/lispy2.html)_.
5. John McCarthy, _[Recursive Functions of Symbolic Expressions and Their Computation by Machine, Part I](http://www-formal.stanford.edu/jmc/recursive/recursive.html)_.
6. Paul Graham, _[The Roots of Lisp](http://www.paulgraham.com/rootsoflisp.html)_.
