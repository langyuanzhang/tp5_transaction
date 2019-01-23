# tp5_transaction
tp5事务提交订单


```php
/*
    提交订单
*/
public function place_order()
{
    $d = input('post.');

    //开启事务
    CommodityModel::startTrans();

    $totalPrices = 0;
    $blist = BusinessModel::where('id','eq',$d['business_id'])->find();
    $distribution_fee = $blist['distribution_fee'];

    //查询有商品库存
    foreach ($d['checksCart'] as $k => $v) {
        $where = ['id'=>$v['id']];
        $goodsinfo = CommodityModel::lock(true)->where($where)->find(); //查询商品

        if(empty($goodsinfo)){
            return $this->retdata(0);
        }
    
        // $total = $goodsinfo['store']; //商品库存
        // if($total < $v['num'] ){
        // 	return $this->retdata(0,array('store'=>'库存不足!'));
        // }

        $totalPrices = floatval($totalPrices) + floatval($goodsinfo['price']);
        $res =  1;
    }

    // //减库存
    // foreach ($d['checksCart'] as $k => $v) {
        
    // 	$where = ['id'=>$v['id']];

    // 	$goodsinfo = CommodityModel::lock(true)->where($where)->find(); //查询商品 

    // 	//$totals = $goodsinfo['store']; //商品库存
    // 	$sell = $goodsinfo['sell']; //商品销量

    // 	//$data['store'] = $totals - $v['num'];

    // 	$data['sell'] = $sell + 0; //$v['num'];

        

    // 	$res = CommodityModel::where('id','eq',$v['id'])->update($data);


    // }

    if($res){

        $order_number = date('YmdHis',time()).rand(1000,9999);

        $orderdata = ['business_id'=>$d['business_id'],'dispatching_id'=>0,'order_number'=>$order_number,'user_id'=>$d['user_id'],'price'=>$d['subtotal'],'distribution_fee'=>$d['distribution_fee'],'form_id'=>$d['formid'],'recipients'=>$d['realname'],'phone'=>$d['phone'],'address'=>$d['area'].$d['area_detail'],'submit_time'=>0];
        
        $order_res = OrderFormModel::insert($orderdata); //生成订单

        if($order_res){

            //订单中间表
            foreach ($d['checksCart'] as $k => $v) {
                
                $middledata = ['commodity_id'=>$v['id'],'order_number'=>$order_number,'business_id'=>$d['business_id'],'name'=>$v['name'],'numbers'=>$v['num'],'price'=>$v['price']];

                Db::name('order_form_middle')->insert($middledata);

                //修改购物车状态
                if($v['id']){
                    $upcart = ['id'=>$v['id'],'status'=>1];
                    ShoppingTrolleyModel::update($upcart);
                }
            }

            CommodityModel::commit(); //提交事务
            return ret_json(200 ,'提示', array('order_number'=>$order_number,'totalPrices'=>floatval($totalPrices) + floatval($distribution_fee),'user_id'=>$d['user_id']) ,  2000,'success');
        }else{
            CommodityModel::rollback(); // 回滚事务
            return ret_json(400 ,'提示', '',  2000,'loading');
        }
    }else{
        CommodityModel::rollback(); // 回滚事务
        return $this->retdata($res);
    }
    exit;
}
```