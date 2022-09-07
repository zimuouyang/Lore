```dart
class FormUtil {
  static Widget textField(
    String formKey,
    String value, {
    TextInputType keyboardType = TextInputType.text,
    FocusNode focusNode,
    controller: TextEditingController,
    onChanged: Function,
    String hintText,
    IconData prefixIcon,
    onClear: Function,
    bool obscureText = false,
    height = 50.0,
    margin = 10.0,
  }) {
    return Container(
      height: height,
      margin: EdgeInsets.all(margin),
      child: Column(
        children: [
          TextField(
              keyboardType: keyboardType,
              focusNode: focusNode,
              obscureText: obscureText,
              controller: controller,
              decoration: InputDecoration(
                hintText: hintText,
                icon: Icon(
                  prefixIcon,
                  size: 20.0,
                ),
                border: InputBorder.none,
                suffixIcon: GestureDetector(
                  child: Offstage(
                    child: Icon(Icons.clear),
                    offstage: value == null || value == '',
                  ),
                  onTap: () {
                    onClear(formKey);
                  },
                ),
              ),
              onChanged: (value) {
                onChanged(formKey, value);
              }),
          Divider(
            height: 1.0,
            color: Colors.grey[400],
          ),
        ],
      ),
    );
  }
}


```

