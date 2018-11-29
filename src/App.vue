<template>
    <div>
        <v-text-field class="ma-3" v-model="expression" style="font-size: 20pt; font-family: Consolas"
                      outline></v-text-field>
        x:
        <v-text-field style="width: 60px; display:inline-block" v-model="variables.x"></v-text-field>
        <div style="width: 100%">
            <v-btn @click="evaluate" style="transform:translateX(-50%); margin-left: 50%">evaluate</v-btn>
        </div>
        <v-text-field v-model="res" class="ma-3" outline></v-text-field>
    </div>
</template>

<script lang="ts">
    Array.prototype.insert = function (index, item) {
        this.splice(index, 0, item);
    };

    import {Vue, Component, Watch} from 'vue-property-decorator'

    enum OpType {Single, Bin, Prefix}

    @Component
    export default class App extends Vue {
        expression: string = '2+x-y^7';//2sin(2+4pi)/3+2x|5/3|+2^(12)
        res: string = '';

        isDigit(c) {
            return c >= '0' && c <= '9'
        }

        isSumb(c) {
            return typeof(c) == "string" && c >= 'a' && c <= 'z';
        }

        variables = {x: 12, y: 2};

        lexer(exp: Function | string): (Function | string)[] {
            let ops = <string[]>[];
            let regex = /(([0-9.]+([eE][0-9])?)|\(|\)|\[|\]|[+*\-\/^]+|[a-z]+)/g;
            let m = undefined;
            do {
                m = regex.exec(this.expression);
                if (m)
                    ops.push(m[0])
            } while (m);
            return ops
        }

        foreachAll(ops: (Function | string)[], type: OpType, predicate, func) {
            let index = ops.findIndex(predicate);
            while (index != -1) {
                let l;
                let r;
                let cur = ops[index];
                console.log('found', cur);
                if (type == OpType.Bin || type == OpType.Prefix) {
                    if (type == OpType.Bin) {
                        l = this.makeF([ops[index - 1]]);
                        if (l == undefined) {
                            type = OpType.Prefix
                        }
                    }
                    r = this.makeF([ops[index + 1]]);
                }
                console.log('before', ops);
                switch (type) {
                    case OpType.Bin:
                        ops.splice(index - 1, 3);
                        index -= 1;
                        break;
                    case OpType.Prefix:
                        ops.splice(index, 2);
                        break;
                    default:
                        ops.splice(index, 1);
                        break;
                }
                console.log('after cut', ops);
                ops.splice(index - (OpType == OpType.Bin ? 1 : 0), 0, func(cur, index, l, r));
                console.log('after insert', ops);
                index = ops.findIndex(predicate);
            }
            return ops;
        }

        fetchBrackets(ops: any[], open, close, action): (Function | string)[] {
            console.log('brackets', ops);
            let i = ops.indexOf(open);
            if (i != -1) {
                let n = 1;
                do {
                    let start = i;
                    while (n != 0 && i < ops.length) {
                        if (ops[i] == open)
                            n++;
                        else if (ops[i] == close)
                            n--;
                        i++;
                    }
                    ops = action(ops, start, i);

                } while (i < ops.length);
            }
            return ops;
        }

        makeF(ops: (Function | String)[]): Function {
            console.log('make', ops);

            ops = this.fetchBrackets(ops, '(', ')', (ops: any[], l, r) => {
                let br = ops.slice(l + 1, r - 1);
                ops.splice(l, r, this.makeF(br));
                return ops;
            });
            ops = this.fetchBrackets(ops, '[', ']', (ops: any[], l, r) => {
                let br = ops.slice(l + 1, r - 1);
                console.log('abs', l, r);
                ops.splice(l, r, (p) => Math.abs(this.makeF(br)(p)));
                return ops;
            });

            ops = this.foreachAll(ops, OpType.Bin, (e) => e == '^', (e, i, l, r) => {
                return (p) => (Math.pow(l(p), r(p))) as Function;
            });
            ops = this.foreachAll(ops, OpType.Bin, (e) => e == '/', (e, i, l, r) => {
                return (p) => (l(p) / r(p)) as Function;
            });
            ops = this.foreachAll(ops, OpType.Bin, (e) => e == '*', (e, i, l, r) => {
                return (p) => (l(p) * r(p)) as Function;
            });
            ops = this.foreachAll(ops, OpType.Bin, (e) => e == '+', (e, i, l, r) => {
                return (p) => (l(p) + r(p)) as Function;
            });
            ops = this.foreachAll(ops, OpType.Bin, (e) => e == '-', (e, i, l, r) => {
                if (l == undefined || l == '[' || l == '(') {
                    return (p) => (-r(p))
                }
                return (p) => (l(p) - r(p)) as Function;
            });
            ops = this.foreachAll(ops, OpType.Single, (e) => this.isDigit(e), (e) => {
                let d = e as string;
                if (d.indexOf('.') != -1) {
                    e = parseFloat(d)
                } else {
                    e = parseInt(d)
                }
                return () => e;
            });
            ops = this.foreachAll(ops, OpType.Single, (e) => this.isSumb(e), (e, i, l, r) => {
                return (p) => (p[e]) as Function;
            });
            return ops[0] as Function;
        }

        evaluate() {
            let lexems = this.lexer(this.expression);

            console.log(lexems);
            let fun = this.makeF(lexems);
            console.log(this.variables)
            this.res = fun(this.variables)
        }
    }
</script>
