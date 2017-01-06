---
title: Java使用socket网络编程实现多人聊天室
date: 2016-01-21 19:52
tags: Java
categories: Java
---

前言：套接字(socket)编程能够实现服务器和客户端的通信，以下通过Socket编程结合多线程实现多人聊天室。
程序展示：
![这里写图片描述](http://img.blog.csdn.net/20160121195425352)![这里写图片描述](http://img.blog.csdn.net/20160121195447460)
![这里写图片描述](http://img.blog.csdn.net/20160121195510914)

<!--more-->
## 界面类 ##
1.客户端界面 ClientView.java

```
public class ClientView extends JFrame implements ActionListener, KeyListener, Runnable {
	private JTextArea textArea;
	private JTextField textField, tfName;
	private JButton btnSend, btnId;
	private JLabel label;
	private JPanel jp1, jp2;
	public boolean isConnect = false;
	private Socket socket = null;
	private DataInputStream inputStream = null;
	private DataOutputStream outputStream = null;
	private JScrollPane scrollPane;
	private static ClientView view;
	
	public JTextArea getTextArea() {
		return textArea;
	}
	
	public DataInputStream getInputStream() {
		return inputStream;
	}
	public DataOutputStream getOutputStream() {
		return outputStream;
	}
	
	public static void main(String[] args) {
		view = new ClientView();
		ServiceView.clientViews.add(view);
		Thread thread = new Thread(view);
		thread.start();
	}
	
	public ClientView() {
		initView();
		try {
			socket = new Socket("localhost", 9090);//连接本地服务器
			
		} catch (UnknownHostException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	
	private void initView() {
		textArea = new JTextArea(20, 20);
		textArea.setEditable(false);
		scrollPane = new JScrollPane(textArea);
		textField = new JTextField(15);
		textField.addKeyListener(this);
		btnSend = new JButton("发送");
		btnSend.addActionListener(this);
		label = new JLabel("昵称：");
		tfName = new JTextField(8);
		jp1 = new JPanel();
		jp2 = new JPanel();
		jp1.add(label);
		jp1.add(tfName);
		tfName.setText("用户0");
		jp1.setLayout(new FlowLayout(FlowLayout.CENTER));
		jp2.add(textField);
		jp2.add(btnSend);
		jp2.setLayout(new FlowLayout(FlowLayout.CENTER));
		
		add(jp1, BorderLayout.NORTH);
		add(scrollPane, BorderLayout.CENTER);
		add(jp2, BorderLayout.SOUTH);
		setTitle("聊天室");
		setSize(500, 500);
		setLocation(450, 150);
		setVisible(true);
		setDefaultCloseOperation(EXIT_ON_CLOSE);
		addWindowListener(new WindowAdapter() { //窗口关闭后断开连接
			@Override
			public void windowClosing(WindowEvent e) {
				try {
					if (socket != null)
						socket.close();
					if (inputStream!= null)
						inputStream.close();
					if (outputStream != null)
						outputStream.close();
				} catch (IOException e1) {
					e1.printStackTrace();
				}
			}
		});
	}

	@Override
	public void actionPerformed(ActionEvent e) {
		if (e.getSource() == btnSend) {
			sendMsg();
		}
	}
	
	private void sendMsg() {
		try {
			String s = textField.getText();
			if (!s.equals("")) { //发送数据
				textField.setText("");
				textArea.append("我(" + tfName.getText() +  "):\r\n" + s + "\r\n");
				outputStream = new DataOutputStream(socket.getOutputStream());
				outputStream.writeUTF(tfName.getText() + "#" + s); 
				outputStream.flush();
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
		
	}

	
	
	@Override
	public void keyPressed(KeyEvent arg0) {
		if (arg0.getKeyCode() == KeyEvent.VK_ENTER) {
			sendMsg();
		}
	}

	@Override
	public void keyReleased(KeyEvent arg0) {
		
	}

	@Override
	public void keyTyped(KeyEvent arg0) {
		
	}
	
	@Override
	public void run() {
		try {
			inputStream = new DataInputStream(socket.getInputStream());
			
			while (true) {
				String[] s = inputStream.readUTF().split("#");
				textArea.append(s[0] + ":\r\n" + s[1] + "\r\n");
			}
		} catch (IOException e) {
		}
		
	}
}
```
2.服务器界面 ServiceView.java

```
public class ServiceView extends JFrame implements ActionListener{
	private JButton btnOpen, btnStop;
	private JLabel label;
	private Service service = null;
	public static ArrayList<ClientView> clientViews = new ArrayList<>();
	private static ServiceView view;
	
	public static ServiceView getView() {
		return view;
	}
	public static void main(String[] args) {
		view = new ServiceView();
	}
	
	public ServiceView() {
		initView();
	}
	
	private void initView() {
		btnOpen = new JButton("打开服务器");
		btnStop = new JButton("关闭服务器");
		btnStop.setEnabled(false);
		btnOpen.addActionListener(this);
		btnStop.addActionListener(this);
		label = new JLabel("服务器停止工作");
		add(label);
		add(btnOpen);
		add(btnStop);
		setTitle("服务器");
		setLayout(new GridLayout(3, 1, 0, 10));
		setSize(300, 300);
		setLocation(450, 150);
		setVisible(true);
		setDefaultCloseOperation(EXIT_ON_CLOSE);
	}
	
	@Override
	public void actionPerformed(ActionEvent e) {
		if (e.getSource() == btnOpen) {
			open();
		} else {
			stop();
		}
	}
	
	public void open() { //开启服务器
		service = new Service();
		Thread thread = new Thread(service);
		thread.start();
		label.setText("服务器正在运行");
		btnOpen.setEnabled(false);
		btnStop.setEnabled(true);
	}
	
	public void stop() { //关闭服务器
		label.setText("服务器已关闭");
		btnOpen.setEnabled(true);
		btnStop.setEnabled(false);
		try {
			synchronized (ClientMannager.sockets) { //关闭各个连接
				for (ChatSocket socket : ClientMannager.sockets) {
					socket.getInputStream().close();
					socket.getOutputStream().close();
				}
				ClientMannager.sockets.removeAllElements();
			}
			
			
			for (ClientView view : clientViews) {
				view.getInputStream().close();
				view.getOutputStream().close();
			}
			
			service.getServerSocket().close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	
}
```

## 功能类 ##
1.ChatSocket.java

```
/*使用socket获得数据流，达到传输数据的目的*/
public class ChatSocket implements Runnable{
	private Socket socket = null;
	private DataInputStream inputStream = null;
	private DataOutputStream outputStream = null;
	
	public DataInputStream getInputStream() {
		return inputStream;
	}
	
	public DataOutputStream getOutputStream() {
		return outputStream;
	}
	
	public ChatSocket(Socket socket) {
		this.socket = socket;
		try {
			inputStream = new DataInputStream(socket.getInputStream());
			outputStream = new DataOutputStream(socket.getOutputStream());
		} catch (IOException e) {
			e.printStackTrace();
		}
		
	}
	
	
	public void send(String send) { //向客户端发送数据
		try {
			outputStream.writeUTF(send);
			outputStream.flush();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	
	@Override
	public void run() { //循环读取客户端发来的数据
		String accept = null;
		
		while (true) {
			try {
				accept = inputStream.readUTF();
				
				ClientMannager.sendAll(this, accept);
			} catch (IOException e) {
				ClientMannager.sockets.remove(this);
			}
		}
	}
}
```
2.ClientMannager.java

```
/*客户端管理器*/
public class ClientMannager {

	private ClientMannager() {
	}
	
	public static Vector<ChatSocket> sockets = new Vector<>();
	
	//向其他客户端发送数据
	public static void sendAll(ChatSocket chatSocket, String send) {
		for (ChatSocket socket : sockets) {
			if (!chatSocket.equals(socket)) {
				socket.send(send);
			}
		}
	}
}
```
3.Service.java

```
/*服务器端，使用线程达到循环等待连接的目的*/
public class Service implements Runnable{

	private ServerSocket serverSocket = null;
	
	public ServerSocket getServerSocket() {
		return serverSocket;
	}
	
	@Override
	public void run() {
		try {
			serverSocket = new ServerSocket(9090); //创建端口
			while (true) { //循地接收客户端的连接
				Socket socket = serverSocket.accept();
				JOptionPane.showMessageDialog(ServiceView.getView(), "客户端连接端口", "TIP", JOptionPane.INFORMATION_MESSAGE);
				ChatSocket chatSocket = new ChatSocket(socket); //新客户端连接
				ClientMannager.sockets.add(chatSocket); //往客户端管理器里添加客户
				Thread thread = new Thread(chatSocket); //启用线程使服务器开始不断接收客户端信息
				thread.start();
				
			}
		} catch (IOException e) {
			e.printStackTrace();
			System.out.println("服务器关闭");
		}
	}
	
}
```