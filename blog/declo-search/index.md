# Searching for Declo

In this project, we try to find `declo` a source-to-source transpiler for python to javascript, using LLMs. 

All this started because I was unhappy with python's functional syntax, and found javascript and rust syntax to be quite functional and declarative. I set out to create a transpiler for converting Python to JS and vice versa. I was hoping to use this tool to write code in JS and have it convert to Python that I could then paste in my editor. 

I envisioned a VSCode Extension / LSP that would understand this merger of Python and JS and compile it down to pure Python. 

All this, so that I can leverage a python runtime with a javascript source code, because I find javascript more declarative and algebraic. 

Which would you prefer?
```py
list(map(lambda x: x + 1, [1, 2, 3]))
```

or 
```js
[1, 2, 3].map(x => x + 1)
```

I like chaining, and wish more things in python natively supported chaining like dataframes in pandas or tensors in pytorch. 



I collected a few input <> output examples and set out to build a transpiler. I started with converting arrow funcs to lambdas:
```py
def arrow_func(js_code = "x => x + 1"):
    parts = js_code.replace(" ", "").split("=>")

    if len(parts) != 2:
        raise ValueError("Invalid arrow_func expression format")

    param, body = parts
    py_code = f"lambda {param}: {body}"
    code_obj = compile(py_code, "<string>", "eval")

    return eval(code_obj)
``` 

but quickly realised that this would get cumbersome with arcane lexical analysis. Maybe a better way is to transform at the AST level. This way I could deal with more structured information and mapping, but this also meant that I could create Python objects at run time and not patch the source. I could always try dis-assembly of the code for better documentation. The AST converter for:

```py
def ast_js2py(js_node: Union[Program, ExpressionStatement, ArrowFunctionExpression, BinaryExpression, Identifier, Literal]) -> ast.AST:
    if isinstance(js_node, Program):
        return ast.Module(body=[ast_js2py(stmt) for stmt in js_node.body], type_ignores=[])
    elif isinstance(js_node, ExpressionStatement):
        return ast.Expr(value=ast_js2py(js_node.expression))
    elif isinstance(js_node, ArrowFunctionExpression):
        return ast.Lambda(
            args=ast.arguments(
                posonlyargs=[],
                args=[ast.arg(arg=param.name, annotation=None, type_comment=None) for param in js_node.params],
                vararg=None,
                kwonlyargs=[], kw_defaults=[], kwarg=None, defaults=[]
            ),
            body=ast_js2py(js_node.body)
        )
    elif isinstance(js_node, BinaryExpression):
        return ast.BinOp(
            left=ast_js2py(js_node.left),
            op=ast.Add() if js_node.operator == "+" else None,  # Extend this for other operators
            right=ast_js2py(js_node.right)
        )
    elif isinstance(js_node, Identifier):
        return ast.Name(id=js_node.name, ctx=ast.Load())
    elif isinstance(js_node, Literal):
        return ast.Constant(value=js_node.value, kind=None)
    else:
        raise ValueError(f"Unsupported JS AST node type: {js_node.__class__.__name__}")
```


This too was riddled with complexity and my tests were breaking. I was scratching my head trying to find the right program that can solve this for me. 

Then I wondered - could LLMs solve this? Can they iterate over examples, execute code and improve their code ? How close can they come to being spec compliant? Obviously, the problem is ill defined because python and JS just have very different specs. 


- Future:
    - Program Search:
        - inputs ... ouputs
        - find the transformation
        - differentiable code search
        - tree search

    - code assistant
    - a compiler 
    - searching for a formal spec transformations

    - define spec compliacne // heuristic
    - make a plan and move in some direction ?? somehow do some optimization
    - somehow needs to acquire data. 



    - custom python with let bindings >> product packaging
    q: i want a python that can break arbitrary for loops and label them. 
    a: here you go: ... edit the grammar asdl ... 


Instead of being completely javascript compliant, I can always choose a subset of its grammar. That is indeed what is declo, a compromise and a testbed for javascript.

1. Prompt Tuning

Let us start with finding a function

 the LLM to create a converter that can convert  



 It returns this:
 ```python
 import re

def js_to_python(js_code):
    # Transform arrow functions
    js_code = re.sub(r'([a-zA-Z_][a-zA-Z0-9_]*) => ([^{}]*)', r'def \\1():\n    return \\2', js_code)
    js_code = re.sub(r'([a-zA-Z_][a-zA-Z0-9_]*) => {([^}]*)}', r'def \\1():\n    \\2', js_code)
    # Transform named functions
    js_code = re.sub(r'const ([a-zA-Z_][a-zA-Z0-9_]*) = (.+)', r'def \\1:\n    return \\2', js_code)
    # Transform blocks with braces
    js_code = re.sub(r'\{([^}]*)\}', lambda m: '\n    '.join(m.group(1).split(';')) + '\n', js_code)
    # Transform maps
    js_code = re.sub(r'\[([^\]]+)\].map\(([^)]+)\)', r'list(map(lambda \\2, [\1]))', js_code)
    # Transform array literals
    js_code = js_code.replace('[', '[').replace(']', ']')
    return js_code
```

obviously not considering any other JS syntax rules. We now need to manually create a test bench with more cases and then debug!!


Okay let's set up the code interpreters
- metric:
    - if the two programs are equal:
        - passes the same tests 
        - AI says same
        - have the same machine code?! obvisouly
            - same IR
    






Let's ask it to generate a few more. 