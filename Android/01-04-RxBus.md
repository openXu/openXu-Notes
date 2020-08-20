RxBus.get().post("GoodsShelvesList", list);

//库房列表返回
@Subscribe(thread = EventThread.MAIN_THREAD, tags = {@Tag("SelectBjTypeFragmentVM")})
public void rxbusMsg2(ArrayList<BjType> list) {
    responseData(list);
}

