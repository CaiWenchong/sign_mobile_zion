<template>
  <div class="canvas">
    <div class="canvas_header">手写签名</div>
    <div class="canvas_canvas">
      <canvas id="myCanvas" width="370" height="400"></canvas>
    </div>
    <div class="canvas_info">
      <span @click="showColorDiv()">*请在输入框写入您的姓名</span>
    </div>
    <div class="canvas_footer">
      <div @click="clearArea()" class="canvas_footer_re">重写</div>
      <div @click="saveImage()" class="canvas_footer_sa">保存</div>
      <div @click="saveImageInfo()" class="canvas_footer_su">提交</div>
    </div>
    <!-- 颜色选择器 -->
    <div class="canvas_color" v-show="showColor == 2">
      <div>
        <input v-model="strokeStyle" type="color" />
      </div>
    </div>
  </div>
</template>

<script>
import CryptoJS from "crypto-js";
export default {
  name: "Sign",
  props: ["globalData", "url", "actionflow_id"],
  data() {
    return {
      touchPressed: false,
      ctx: null,
      strokeStyle: "#000000", //书写颜色
      lineWidth: 4, //线条宽度
      lastX: null,
      lastY: null,
      canvas: null,
      showColor: 1, //显示颜色
      sColor: 0, //显示颜色
      storeConfig: {
        // zion项目的api地址
        // gql_apiUrl:
        //   "https://zion-app.functorz.com/zero/JmAxbl1kYqo/api/graphql-v2",
          gql_apiUrl:"",
        // zion项目地址对应的authorization
        gql_authorization:
          "Bearer eyJhbGciOiJIUzI1NiJ9.eyJyb2xlcyI6WyJhZG1pbiJdLCJaRVJPX1VTRVJfSUQiOiIxMDA5OTk5OTk5OTk5OTk5IiwiZGVmYXVsdFJvbGUiOiJhZG1pbiIsImhhc3VyYV9jbGFpbXMiOnsieC1oYXN1cmEtdXNlci1pZCI6IjEwMDk5OTk5OTk5OTk5OTkiLCJ4LWhhc3VyYS1hbGxvd2VkLXJvbGVzIjpbImFkbWluIl0sIngtaGFzdXJhLWRlZmF1bHQtcm9sZSI6ImFkbWluIn0sInplcm8iOnt9fQ.x0qj8zJQgzhk55rDbeosmSRe5hSie15rXHkt4WNyWAc",

        is_multi_app: false,
        default_system_system: null,
        system_model_list: [],
        // actionflowmain_id: "5e5e155d-7b3c-4771-bf94-ceb597cbb666",
        actionflowmain_id:"",
        token: "", //调试token
        env: "H5", //1.MP-WEIXIN 2.H5
      },
    };
  },
  mounted() {
    console.log("props:", this.$props);

    if (!this.$props.url && !this.$props.actionflow_id) {
      console.log("url或者actionflow_id为空");
    }else{
      this.storeConfig.gql_apiUrl=this.$props.url 
      this.storeConfig.actionflowmain_id=this.$props.actionflow_id
    }

  
    this.$nextTick(() => {
      let canvas = document.getElementById("myCanvas");
      this.canvas = canvas;
      this.ctx = canvas.getContext("2d");
      this.Init();
    });
  },

  methods: {
    // 封装请求函数,这里的请求默认是在uniapp环境
    async request(options) {
      process.env.NODE_ENV !== "production" &&
        console.log(
          "%crequest发起请求传入options：",
          "color: #20b9be",
          options
        );
      return await new Promise((resolve, reject) => {
        if (this.storeConfig?.env == "MP-WEIXIN") {
          wx.request(options)
            .then((res) => {
              resolve(res.data);
            })
            .catch((e) => {
              reject(e);
            });
        } else if (this.storeConfig?.env == "H5") {
          fetch(options.url, {
            method: options?.method,
            body: JSON.stringify(options?.data),
            headers: {
              ...options?.header,
              "Content-Type": "application/json",
            },
          })
            .then((res) => {
              resolve(res.json());
            })
            .catch((e) => {
              reject(e);
            });
        }
      });
    },

    async local_umedia(media_url, url = "", header_authorization = "") {
      // 本地上传，公网链接上传
      let variables = {};
      let gql = this.get_mutation_gql({
        operation: `umedia_${media_url}`,
      });
      return await this.request({
        url: url || this.storeConfig?.gql_apiUrl,
        header: {
          authorization:
            header_authorization || this.storeConfig?.gql_authorization,
        },
        method: "POST",
        data: {
          variables,
          query: gql,
        },
      }).then((res) => {
        let response = res?.data?.response?.umedia_result;
        if (!response) {
          throw res;
        } else {
          return response;
        }
      });
    },

    get_mutation_gql(data = {}, mode = 1, response_key = "response") {
      let {
        operation = "",
        field_string = ``,
        where = {},
        _set,
        _inc,
        objects = [], //[]或{}
        object = {},
        args,
        on_conflict,
        pk_columns,
        params,
        id,
      } = data;
      let response_body = ``;
      let gql = ``;

      let op_type = operation.slice(0, 6);
      if (
        op_type != "update" &&
        op_type != "delete" &&
        op_type != "insert" &&
        op_type != "action" &&
        op_type != "umedia"
      ) {
        throw "operation不正确，可选值：1.update 2.delete 3.insert 4.action 5.umedia";
      }
      if (op_type == "others") {
        let model = operation.slice(7);
        let params_string = "";
        if (
          typeof params == "object" &&
          params &&
          Object.keys(params).length > 0
        ) {
          let tempStr = this.gql_string(params);
          params_string = `${tempStr.slice(1, tempStr.length - 1)}`;
        }
        let queryResponse = "";
        if (field_string) {
          queryResponse = ` {
			  ${field_string}
			}`;
        }
        let queryBody = "";
        if (params_string) {
          queryBody = `(${params_string})`;
        }
        response_body = `${response_key}:${model}${queryBody}${queryResponse}`;
        gql = `mutation fz_others { ${response_body} }`;
        return mode === 1 ? gql : mode === 2 ? response_body : "";
      }
      if (op_type == "action") {
        let actionflow_id = operation.slice(7);
        let args_string = `${this.gql_string(args || {})}`;
        response_body = `${response_key}: fz_invoke_action_code(
      testPassword: "doushizhutou3"
      args: { actionflow_id: "${actionflow_id}", actionflow_data: ${args_string} }
      jsCode: "const actionflow_data = context.getArg(\\"actionflow_data\\") || {};const actionflow_id = context.getArg(\\"actionflow_id\\") || \\"\\";let action_result = context.callActionFlow(actionflow_id, null, actionflow_data);context.setReturn(\\"action_result\\", action_result);"
      updateDb: true
      accountId: 1000000000000001
    )`;
        gql = `mutation fz_action {
      ${response_body}
    }`;
        return mode === 1 ? gql : mode === 2 ? response_body : "";
      }

      if (op_type == "umedia") {
        let media_url = operation.slice(7);
        response_body = `${response_key}: fz_invoke_action_code(
      testPassword: "doushizhutou3"
      args: { media_url: "${media_url}" }
      jsCode: "const media_url = context.getArg(\\"media_url\\") || \\"\\";let umedia_result = context.uploadMedia(media_url, {});context.setReturn(\\"umedia_result\\", umedia_result);"
      updateDb: true
      accountId: 1000000000000001
    )`;
        gql = `mutation fz_umedia {
        ${response_body}
      }`;
        return mode === 1 ? gql : mode === 2 ? response_body : "";
      }

      // 自动追加系统配置
      if (op_type == "insert" || op_type == "update") {
        let {
          is_multi_app: is_multi_app_tmp,
          default_system_system: default_system_system_tmp = null,
          system_model_list: system_model_list_tmp = [],
        } = this.storeConfig;
        let model = operation.slice(7);
        system_model_list_tmp.forEach((item) => {
          let reg = new RegExp(`^${item}`);
          if (reg.test(model) && is_multi_app_tmp) {
            // update时
            if (!where?.system_system) {
              where.system_system = {
                _eq: default_system_system_tmp,
              };
            }
            // insert时
            objects.forEach((itemObject) => {
              if (!itemObject?.system_system) {
                itemObject.system_system = default_system_system_tmp;
              }
            });
            // insert_xxx_one时
            if (!object?.system_system) {
              object.system_system = default_system_system_tmp;
            }
          }
        });
      }

      let where_string = "";
      if (typeof where == "object" && where && Object.keys(where).length > 0) {
        where_string = `where:${this.gql_string(where)}`;
      }

      let _set_string = "";
      if (typeof _set == "object" && _set && Object.keys(_set).length > 0) {
        _set_string = `_set:${this.gql_string(_set)}`;
      }

      let _inc_string = "";
      if (typeof _inc == "object" && _inc && Object.keys(_inc).length > 0) {
        _inc_string = `_inc:${this.gql_string(_inc)}`;
      }

      let objects_string = "";
      if (
        typeof objects == "object" &&
        objects &&
        Object.keys(objects).length > 0
      ) {
        objects_string = `objects:${this.gql_string(objects, 3)}`;
      }

      let on_conflict_string = "";
      if (
        typeof on_conflict == "object" &&
        on_conflict &&
        Object.keys(on_conflict).length > 0
      ) {
        on_conflict_string = `on_conflict:${this.gql_string(on_conflict, 3)}`;
      }

      let object_string = "";
      if (
        typeof object == "object" &&
        object &&
        Object.keys(object).length > 0
      ) {
        object_string = `object:${this.gql_string(object, 3)}`;
      }

      let pk_columns_string = "";
      if (
        typeof pk_columns == "object" &&
        pk_columns &&
        Object.keys(pk_columns).length > 0
      ) {
        pk_columns_string = `pk_columns:${this.gql_string(pk_columns)}`;
      }

      let id_string = "";
      if (typeof id != "undefined") {
        id_string = `id:${id}`;
      }

      let responseBodyAutoAttach = "id";
      if (/insert_\w+_one/.test(operation)) {
        response_body = `${response_key}:${operation}(
		  ${object_string}
		  ${on_conflict_string}
		) {
		    ${responseBodyAutoAttach}
		    ${field_string}
		}`;
      } else if (/update_\w+_by_pk/.test(operation)) {
        response_body = `${response_key}:${operation}(
		  ${_set_string}
		  ${_inc_string}
		   ${pk_columns_string}
		) {
		    ${responseBodyAutoAttach}
		    ${field_string}
		}`;
      } else if (/delete_\w+_by_pk/.test(operation)) {
        response_body = `${response_key}:${operation}(
		  ${id_string}
		) {
		    ${responseBodyAutoAttach}
		    ${field_string}
		}`;
      } else {
        response_body = `${response_key}:${operation}(
		  ${where_string}
		  ${_set_string}
		  ${_inc_string}
		  ${objects_string}
		  ${on_conflict_string}
		) {
		  affected_rows
		  returning{
		    ${responseBodyAutoAttach}
		    ${field_string}
		  }
		}`;
      }
      gql = `mutation ${operation} {
    ${response_body}
  }`;
      return mode === 1 ? gql : mode === 2 ? response_body : "";
    },

    async mutation(data, url = "", header_authorization = "") {
      let variables = {};
      let gql = this.get_mutation_gql(data);
      return await this.request({
        url: url || this.storeConfig?.gql_apiUrl,
        header: {
          authorization:
            header_authorization || this.storeConfig?.gql_authorization,
        },
        method: "POST",
        data: {
          variables,
          query: gql,
        },
      }).then((res) => {
        let response = res?.data?.response;
        if (!response) {
          throw res;
        } else {
          return response;
        }
      });
    },

    gql_string(obj, mode = 1) {
      if (mode === 1) {
        return JSON.stringify(obj).replace(/"(\w+?)":/g, "$1:");
      } else if (mode === 2) {
        return JSON.stringify(obj).replace(/"/g, "");
      } else if (mode === 3) {
        return JSON.stringify(obj)
          .replace(/"(\w+?)":/g, "$1:")
          .replace(
            /([,\{])update_columns:"([\w,\[\]]+)"([,\}])/g,
            "$1update_columns:$2$3"
          )
          .replace(
            /([,\{])constraint:"([\w,\[\]]+)"([,\}])/g,
            "$1constraint:$2$3"
          );
      } else if (mode === 4) {
        return obj
          .replace(/\\/g, "\\\\")
          .replace(/\r\n/g, "\\n")
          .replace(/\"/g, '\\"')
          .replace(/\n/g, "\\n");
      }
    },

    // 保存服务器
    saveImageInfo() {
      let b = this.canvas.toDataURL();
      this.local_umedia(b).then((imageRes) => {
        if (imageRes.id) {
          this.insertSignPictureOne(imageRes.id).then((res) => {
            if (res.id) {
              this.$props.globalData.sign_image_id = imageRes.id;
            }
          });
        }
      });
      console.log("props:", this.$props);
    },

    insertSignPictureOne(signImageId) {
      let params = {
        operation: "insert_sign_picture_one",
        object: {
          pic_url_id: signImageId,
          pic_id: signImageId,
        },
        isloading: false,
        field_string: `id pic_id pic_url{id url} `,
      };
      return this.mutation(params);
    },

    Init() {
      // 移动前
      this.canvas.addEventListener(
        "touchstart",
        (event) => {
          if (event.targetTouches.length == 1) {
            event.preventDefault(); // 阻止浏览器默认事件，重要
            var touch = event.targetTouches[0];
            this.touchPressed = true;
            this.draw(
              touch.pageX - this.canvas.offsetLeft,
              touch.pageY - this.canvas.offsetTop,
              false
            );
          }
        },
        false
      );
      // 移动中
      this.canvas.addEventListener(
        "touchmove",
        (event) => {
          if (event.targetTouches.length == 1) {
            event.preventDefault(); // 阻止浏览器默认事件，重要
            var touch = event.targetTouches[0];
            if (this.touchPressed) {
              this.draw(
                touch.pageX - this.canvas.offsetLeft,
                touch.pageY - this.canvas.offsetTop,
                true
              );
            }
          }
        },
        false
      );
      // 移动结束
      this.canvas.addEventListener(
        "touchend",
        (event) => {
          if (event.targetTouches.length == 1) {
            event.preventDefault(); // 阻止浏览器默认事件，防止手写的时候拖动屏幕，重要
            this.touchPressed = false;
          }
        },
        false
      );
    },
    draw(x, y, isDown) {
      let ctx = this.ctx;
      if (isDown) {
        ctx.beginPath();
        ctx.strokeStyle = this.strokeStyle;
        ctx.lineWidth = this.lineWidth;
        ctx.lineJoin = "round";
        ctx.moveTo(this.lastX, this.lastY);
        ctx.lineTo(x, y);
        ctx.closePath();
        ctx.stroke();
      }
      this.lastX = x;
      this.lastY = y;
    },
    // 重写
    clearArea() {
      this.ctx.setTransform(1, 0, 0, 1, 0, 0);
      this.ctx.clearRect(0, 0, this.ctx.canvas.width, this.ctx.canvas.height);
    },
    // 保存本地
    saveImage() {
      let a = document.createElement("a");
      a.href = this.canvas.toDataURL();
      a.download = "sign";
      a.click(); //保存
    },

    // 弹出颜色
    showColorDiv() {
      return;
      this.sColor += 1;
      if (this.sColor == 10) {
        this.showColor = 2;
      }
    },
    //签完名的图片旋转处理
    // src为你、base64编码；edg为角度，0-360；callback回调
    rotateBase64Img(src, edg, callback) {
      var canvas = document.createElement("canvas");
      var ctx = canvas.getContext("2d");
      var imgW; //图片宽度
      var imgH; //图片高度
      var size; //canvas初始大小
      if (edg % 90 != 0) {
        console.error("旋转角度必须是90的倍数!");
        throw "旋转角度必须是90的倍数!";
      }
      edg < 0 && (edg = (edg % 360) + 360);
      const quadrant = (edg / 90) % 4; //旋转象限
      const cutCoor = { sx: 0, sy: 0, ex: 0, ey: 0 }; //裁剪坐标
      var image = new Image();
      image.crossOrigin = "anonymous";
      image.src = src;
      image.onload = function () {
        imgW = image.width;
        imgH = image.height;
        size = imgW > imgH ? imgW : imgH;
        canvas.width = size * 2;
        canvas.height = size * 2;
        switch (quadrant) {
          case 0:
            cutCoor.sx = size;
            cutCoor.sy = size;
            cutCoor.ex = size + imgW;
            cutCoor.ey = size + imgH;
            break;
          case 1:
            cutCoor.sx = size - imgH;
            cutCoor.sy = size;
            cutCoor.ex = size;
            cutCoor.ey = size + imgW;
            break;
          case 2:
            cutCoor.sx = size - imgW;
            cutCoor.sy = size - imgH;
            cutCoor.ex = size;
            cutCoor.ey = size;
            break;
          case 3:
            cutCoor.sx = size;
            cutCoor.sy = size - imgW;
            cutCoor.ex = size + imgH;
            cutCoor.ey = size + imgW;
            break;
        }
        ctx.translate(size, size);
        ctx.rotate((edg * Math.PI) / 180);
        ctx.drawImage(image, 0, 0);
        var imgData = ctx.getImageData(
          cutCoor.sx,
          cutCoor.sy,
          cutCoor.ex,
          cutCoor.ey
        );
        if (quadrant % 2 == 0) {
          canvas.width = imgW;
          canvas.height = imgH;
        } else {
          canvas.width = imgH;
          canvas.height = imgW;
        }
        ctx.putImageData(imgData, 0, 0);
        callback(canvas.toDataURL());
      };
    },
  },
};
</script>
<style>
.canvas_header {
  width: 100%;
  text-align: center;
  line-height: 50px;
  font-weight: 800;
  font-size: 20px;
}
.canvas_canvas {
  width: 370px;
  margin: 0 auto;
  border: 1px solid #aaa;
}
.canvas_info {
  margin: 10px 8%;
  font-size: 12px;
}
.canvas_footer {
  width: 80%;
  margin: 10px 10%;
  display: flex;
  justify-content: space-between;
  line-height: 40px;
  text-align: center;
  color: white;
  font-weight: 800;
  padding-bottom: 20px;
}
.canvas_footer_re {
  width: 100px;
  background: #f11919;
  border-radius: 10px;
}
.canvas_footer_sa {
  width: 100px;
  background: #1989f1;
  border-radius: 10px;
}
.canvas_footer_su {
  width: 100px;
  background: #2dcb7a;
  border-radius: 10px;
}
</style>
