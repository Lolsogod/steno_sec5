<script lang="ts">
    import * as twist from '../mersenne-twister.js'
    import * as CryptoJS from 'crypto-js'
    let imgs: FileList;
    let msg: string = "";
    let password: string = "";
    let preview: string;
    const encrypt = () =>{
        load(imgs,download)
    }

    //load image to canvas
    const load = (imgs: FileList,callback: any) =>{
        if (imgs && imgs[0]){
            let img = imgs[0]
            console.log(img)
            let reader = new FileReader()
            reader.onload = (e) =>{
                if (e.target && e.target.result){
                    let data = e.target.result
                    //@ts-ignore
                    preview = e.target.result
                    let image = new Image()
                    image.onload = () => {
                        let maxsize = 500
                        let w=image.width
                        let h=image.height
                        if(maxsize>0){
                            if(w>maxsize){
                                h=h*(maxsize/w);
                                w=maxsize;
                            }
                            if(h>maxsize){
                                w=w*(maxsize/h);
                                h=maxsize;
                            }
                            w=Math.floor(w);
                            h=Math.floor(h);
                        }
                        let canvas = document.createElement('canvas');
                        canvas.id = 'canvas';
                        canvas.width = w;
                        canvas.height = h;
                        canvas.style.display = "none";
                        let body = document.getElementsByTagName("body")[0];
                        body.appendChild(canvas);
                        let context = canvas.getContext('2d');
                        if(context){
                            context.drawImage(image, 0, 0,image.width,image.height,0,0,w,h);
                            callback();
                            document.body.removeChild(canvas);
                        }
                    }
                    image.src = data.toString();
                }
            }
            reader.readAsDataURL(img);
        }
    }
    //download image
    const download = () =>{
        write(msg, password, 1)
        let myCanvas = <HTMLCanvasElement> document.getElementById("canvas");
        if (myCanvas){
            let image = myCanvas.toDataURL("image/png");
            let element = document.createElement('a');
            element.setAttribute('href', image);
            element.setAttribute('download', 'result.png');
            element.style.display = 'none';
            document.body.appendChild(element);
            element.click();
            document.body.removeChild(element);
        }
    }
    //write message to image
    const write = (msg: string, enc_key: string, num_copy: number) =>{
        enc_key=(enc_key === undefined)?'':enc_key;
        num_copy=(num_copy === undefined)?5:num_copy;

        try{
            let c = <HTMLCanvasElement> document.getElementById("canvas");
            let ctx=c.getContext("2d");
            if (ctx){
                let imgData=ctx.getImageData(0,0,c.width,c.height);
                let encode_len = Math.floor(imgData.data.length / 4) * 3;
                // prepare data
                let bit_stream = str_to_bits(msg, num_copy);
                bit_stream = prepare_write_data(bit_stream, enc_key, encode_len);
                write_lsb(imgData, bit_stream);
                ctx.putImageData(imgData,0,0);
                return true; 
            }
        }
        catch(err){
            return err;
        }
    }
    
    const str_to_bits = (str:string, num_copy:number) =>{
        let utf8array=utf8Encode(str);
        let result=Array();
        let utf8strlen=utf8array.length;
        for(let i=0;i<utf8strlen;i++){
            for(let j=128; j>0; j=Math.floor(j/2))
            {
                if(Math.floor(utf8array[i]/j))
                {
                    for(let cp=0; cp<num_copy; cp++) result.push(1);
                    utf8array[i] -=j;
                } else for(let cp=0; cp<num_copy; cp++) result.push(0);
            }
        }
        for(let j=0;j<24;j++) for(let i=0;i<num_copy;i++) {
            result.push(1);
        }
        return result;
    }
    const utf8Encode = (str: string) =>{
        let bytes = [], offset = 0, length, char;

        str = encodeURI(str);
        length = str.length;

        while (offset < length) {
            char = str[offset];
            offset += 1;

            if ('%' !== char) {
            bytes.push(char.charCodeAt(0));
            } else {
            char = str[offset] + str[offset + 1];
            bytes.push(parseInt(char, 16));
            offset += 2;
            }
        }
        return bytes;
    }

    const  prepare_write_data = (data_bits: string | any[], enc_key: string, encode_len: number) =>{
        let data_bits_len = data_bits.length;
        if(data_bits.length > encode_len) throw "Can not hold this many data!";
        let result=Array(encode_len);
        for(let i=0; i<encode_len; i++){
            result[i] = Math.floor(Math.random()*2); 
        }
        let order = get_hashed_order(enc_key, encode_len);
        for(let i=0; i<data_bits_len; i++) result[order[i]] = data_bits[i];

        return result;
    }
    const get_hashed_order = (password: string | CryptoJS.lib.WordArray, arr_len: number) => {
        let orders = Array.from(Array(arr_len).keys());
        let result = [];
        let loc;
        let seed = CryptoJS.SHA512(password).words.reduce((total, num) => {return total + Math.abs(num);}, 0);
        let rnd = new twist.MersenneTwister(seed);
        for(let i=arr_len; i>0; i--){
            loc = rnd.genrand_int32() % i;
            result.push(orders[loc]);
            orders[loc] = orders[i-1];
        }
        return result;
    }
    const write_lsb = (imgData: ImageData,setdata: any[]) =>{
        const unsetbit = (k: number)=>{
            return (k%2==1)?k-1:k;
        }
        const setbit = (k: number) =>{
            return (k%2==1)?k:k+1;
        }
        let j=0;
        for (let i=0;i<imgData.data.length;i+=4)
        {
            imgData.data[i] = (setdata[j])?setbit(imgData.data[i]):unsetbit(imgData.data[i]);
            imgData.data[i+1] = (setdata[j+1])?setbit(imgData.data[i+1]):unsetbit(imgData.data[i+1]);
            imgData.data[i+2] = (setdata[j+2])?setbit(imgData.data[i+2]):unsetbit(imgData.data[i+2]);
            imgData.data[i+3]=255;
            j+=3;
        }
    }

    //decrypt----------------------------------------------------------------------------------------------
    let password2:string = ""
    let imgs2: FileList
    let res: String
    interface DecryptRes{
        status: boolean,
        msg: string
    }
        const decrypt = () =>{
            load(imgs2, read)
        }
        const read = () =>{
            console.log("here")
            let t =from_canvas(password2,1);
            res = t.msg
        }

        const from_canvas = (enc_key: string | undefined, num_copy: number | undefined): DecryptRes =>{
        enc_key=(enc_key === undefined)?'':enc_key;
        num_copy=(num_copy === undefined)?5:num_copy;

        let c, ctx, imgData;

        try{
            c= <HTMLCanvasElement> document.getElementById('canvas');
            ctx=c.getContext("2d");
            if (ctx) imgData=ctx.getImageData(0,0,c.width,c.height);
        }
        catch(err){
            return {status: false,  msg: "ошибка"};
        }

        try{
            if (imgData){
                let bits_stream = get_bits_lsb(imgData);
                bits_stream = prepare_read_data(bits_stream, enc_key);
                let msg = bits_to_str(bits_stream, num_copy);
                if(msg==null) return {status: false, msg: "Неверный пароль"};
                return {status: true,  msg};
            }
        }
        catch(err){
            return {status: false, msg: "Неверный пароль"};
        }
        return {status: false, msg: "ошибка"};
    }
    const get_bits_lsb = (imgData: ImageData) =>{
        let result=Array();
        for (let i=0;i<imgData.data.length;i+=4)
        {
            result.push((imgData.data[i]%2==1)?1:0);
            result.push((imgData.data[i+1]%2==1)?1:0);
            result.push((imgData.data[i+2]%2==1)?1:0);
        }
        return result;
    }
    const prepare_read_data = (data_bits: string | any[], enc_key: string | CryptoJS.lib.WordArray) =>{
        let data_bits_len = data_bits.length;
        let result=Array(data_bits_len);
        let order = get_hashed_order(enc_key, data_bits_len);

        for(let i=0; i<data_bits_len; i++) result[i] = data_bits[order[i]];

        return result;
    }
    const bits_to_str = (bitarray: string | any[], num_copy: number) => {
        const merge_bits = (bits: string | any[]) => {
            let bits_len = bits.length;
            let bits_sum = 0;
            for(let i=0;i<bits_len;i++) bits_sum += bits[i];
            return Math.round(bits_sum / bits_len);
        }

        let msg_array=Array();
        let data, tmp;

        let msg_array_len=Math.floor(Math.floor(bitarray.length/num_copy)/8);
        for(let i=0; i<msg_array_len; i++){
            data = 0;
            tmp = 128;
            for(let j=0; j<8; j++){
                data += merge_bits(bitarray.slice((i*8 + j) *num_copy, (i*8 + j + 1) *num_copy))*tmp;
                tmp = Math.floor(tmp/2);
            }
            if(data == 255) break; 
            msg_array.push(data);
        }

        return utf8Decode(msg_array);
    }
    const utf8Decode = (bytes: string | any[]) => {
        let chars = [], offset = 0, length = bytes.length, c, c2, c3;

        while (offset < length) {
            c = bytes[offset];
            c2 = bytes[offset + 1];
            c3 = bytes[offset + 2];

            if (128 > c) {
            chars.push(String.fromCharCode(c));
            offset += 1;
            } else if (191 < c && c < 224) {
            chars.push(String.fromCharCode(((c & 31) << 6) | (c2 & 63)));
            offset += 2;
            } else {
            chars.push(String.fromCharCode(((c & 15) << 12) | ((c2 & 63) << 6) | (c3 & 63)));
            offset += 3;
            }
        }

        return chars.join('');
    }
</script>
<main>
    <h1>Stenography</h1>
    <h2>Encrypt</h2>
    Image: <input type="file" id="file" accept="image/*" bind:files={imgs} /> 
    <br /><br />
    Password: <input type="text" id="pass" bind:value={password}/> 
    <br /><br />
    Message:<textarea id="msg" bind:value={msg}></textarea> 
    <br /><br />
    <button on:click={encrypt}>Encrypt</button>
    <h2>Decrypt</h2>
    Image: <input type="file" id="file1" accept="image/*" bind:files={imgs2}/> 
    <br /><br />
    Password: <input type="text" id="pass2" bind:value={password2} /> 
    <br /><br />
    <button on:click={decrypt}>Decrypt</button>
    <br>
    <!--<img  src={preview} alt="">-->
    <br /><br />
    Results:
    {#if res}
    <div id="res">{res}</div>  
    {/if}
</main>
<style>
    *{
        font-family: sans-serif;
    }
</style>