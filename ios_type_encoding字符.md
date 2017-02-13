## `objc_property_attribute_t.name[0]` 的type encodings

```c
 static const char XZHPropertyAttributeBegin = 'T';//T@\"NSString\",C,N,V_name，作为属性编码的开始符，不作为属性权限修饰符
static const char XZHPropertyAttributeIvarName = 'V';//V_name，表示Ivar的名字，不作为属性权限修饰符
static const char XZHPropertyAttributeCopy = 'C';
static const char XZHPropertyAttributeCustomGetter = 'G';
static const char XZHPropertyAttributeCustomSetter = 'S';
static const char XZHPropertyAttributeDynamic = 'D';
static const char XZHPropertyAttributeEligibleForGarbageCollection = 'P';
static const char XZHPropertyAttributeNonAtomic = 'N';
static const char XZHPropertyAttributeOldStyle = 't';
static const char XZHPropertyAttributeReadOnly = 'R';
static const char XZHPropertyAttributeRetain = '&';
static const char XZHPropertyAttributeWeak = 'W';
```

## `objc_ivar` 的type encodings

```c
static const char XZHIvarTypeUnKnown = _C_UNDEF;//?
static const char XZHIvarTypeObject = _C_ID;//@
static const char XZHIvarTypeClass = _C_CLASS;//#
static const char XZHIvarTypeSEL = _C_SEL;//:
static const char XZHIvarTypeChar = _C_CHR;//c
static const char XZHIvarTypeUnsignedChar = _C_UCHR;//C
static const char XZHIvarTypeInt = _C_INT;//i
static const char XZHIvarTypeUnsignedInt = _C_UINT;//I
static const char XZHIvarTypeShort = _C_SHT;//s
static const char XZHIvarTypeUnsignedShort = _C_USHT;//S
static const char XZHIvarTypeLong = _C_LNG;//l
static const char XZHIvarTypeUnsignedLong = _C_ULNG;//L
static const char XZHIvarTypeLongLong = 'q';
static const char XZHIvarTypeUnsignedLongLong = 'Q';
static const char XZHIvarTypeFloat = _C_FLT;//f
static const char XZHIvarTypeDouble = _C_DBL;//d
static const char XZHIvarTypeLongDouble = 'D';
static const char XZHIvarTypeBOOL = 'B';
static const char XZHIvarTypeVoid = _C_VOID;//v
static const char XZHIvarTypeCPointer = _C_PTR;//^
static const char XZHIvarTypeCString = _C_CHARPTR;//*
static const char XZHIvarTypeCArray = _C_ARY_B;//[
static const char XZHIvarTypeCArrayEnd = _C_ARY_E;//]
static const char XZHIvarTypeCStruct = _C_STRUCT_B;//{
static const char XZHIvarTypeCStructEnd = _C_STRUCT_E;//}
static const char XZHIvarTypeCUnion = _C_UNION_B;//(
static const char XZHIvarTypeCUnionEnd = _C_UNION_E;//)
static const char XZHIvarTypeCBitFields = _C_BFLD;//b
```

## runtime.h中定义出的type encoding字符

```c
#define _C_ID       '@'
#define _C_CLASS    '#'
#define _C_SEL      ':'
#define _C_CHR      'c'
#define _C_UCHR     'C'
#define _C_SHT      's'
#define _C_USHT     'S'
#define _C_INT      'i'
#define _C_UINT     'I'
#define _C_LNG      'l'
#define _C_ULNG     'L'
#define _C_LNG_LNG  'q'
#define _C_ULNG_LNG 'Q'
#define _C_FLT      'f'
#define _C_DBL      'd'
#define _C_BFLD     'b'
#define _C_BOOL     'B'
#define _C_VOID     'v'
#define _C_UNDEF    '?'
#define _C_PTR      '^'
#define _C_CHARPTR  '*'
#define _C_ATOM     '%'
#define _C_ARY_B    '['
#define _C_ARY_E    ']'
#define _C_UNION_B  '('
#define _C_UNION_E  ')'
#define _C_STRUCT_B '{'
#define _C_STRUCT_E '}'
#define _C_VECTOR   '!'
#define _C_CONST    'r'
```

注意，`'A'`是一个字符，`"A"`才是一个字符串。